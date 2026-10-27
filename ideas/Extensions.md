# Wizard Extensions

A proposal for letting an instruction set extension live outside `src/engine/`: it is switched on at build time, registers its opcodes and behavior through a small set of hooks, and degrades gracefully in tiers that have no fast path for it.
Transactions are the motivating client.
Two small extensions show the mechanism end to end: `undo` (checkpoints for memory 0) and `ministe` (a subset of the Wasm Simple Transactions proposal, STE).

The framework is being built twice, so the two can be compared:

- **Version 1** (this document, branch `ext-v1`) uses only features Virgil has today.
- **Version 2** (branch `ext-v2`, to come) uses open enums with per-case methods, as proposed in titzer/virgil#653. Extension instructions there are cases of `Opcode` itself.

Both versions implement the same two extensions and are measured the same way. Eliot Moss's transactional Wizard (UMass) is a reference for what a real extension needs. None of its code is used.

Status: draft, for discussion.

## Constraints

The design works within these constraints from discussion with Ben (September 2026):

- **No fields added to classes from outside the class.** Initialization order would be unclear, and lazy initialization adds state and the risk of recursive initialization. Weak maps are preferred.
- **No per-case methods on enums for now.**
- **Extensible enums and top-level methods wait** for a stable Virgil language revision. Nothing here depends on them.
- **A configuration phase at build time** in which extensions register their opcodes, validation, JIT support, fast-interpreter handlers and type system changes.
- **Two kinds of opcode claims.** Some extensions need fixed encodings because input modules use a binary format (transactions). Others can take any leftover opcode because the engine only uses them internally (for example, call-optimizing opcodes for interpreter runtimes).
- **Until extensible enums exist,** rely on opcode tables rather than the `Opcode` enum.
- **Visitor extensibility** is an open problem; this proposal suggests an answer.

## What Wizard already provides

- `Opcode` (`src/engine/Opcodes.v3`) is a closed enum, but lookup goes through `OpcodePage` tables per prefix (0xFB to 0xFE), with LEB subpages and valid bitmaps.
- `InternalOpcode` already claims spare bytes (probes at 0x1E, 0x1D, 0x16; crash opcodes at 0xF0, 0xF1). This is the precedent for internal, floating opcodes.
- `Extension` (`src/engine/Extension.v3`) is a closed feature set, gated in the validator by `checkExtension`.
- `SinglePassCompiler` calls `bailout("unsupported opcode")` on an opcode it cannot compile, which sets `success = false`.
- `build.sh --object-model=<dir>` already selects code by directory at build time.
- `StructDecl.infos` and `ArrayDecl.infos` are open chains that object models extend from their own files. `HeapTypeDecl.canon_id` is a global id assigned at canonicalization.
- Probes run Virgil code against a live Wasm frame in every tier, through `FrameAccessor` and, in fast-int, `runtime_PROBE_instr(stack, func, pc)`.
- Virgil runs component initializers at compile time and serializes the result into the binary, so tables built during registration are build-time data.

## Design

### 1. Extension instructions are descriptor objects, one class each

```
class ExtOpcode(prefix: u8, code: u16, mnemonic: string,
                imms: Array<ImmKind>, sig: SigDecl, claim: OpcodeClaim) {
    def validate(v: CodeValidator) -> bool { return true; }       // beyond the signature
    def interpret(f: ExtFrame) -> TrapReason;                     // reference semantics
    def accept(v: BytecodeVisitor) { v.visitExt(this); }
    def opensBlock() -> bool { return false; }                    // block-shaped (section 9)
    def onEnd(v: CodeValidator, pos: int) { }
    def onBranchOut(v: CodeValidator) -> bool { return true; }
    def endsBlock() -> bool { return false; }
}
enum OpcodeClaim { FIXED, FLOATING }
```

These are the same fields as `Opcode`, plus a claim; `owner` is set when the opcode is placed.
Each instruction is a small subclass that overrides the hooks it needs, so its encoding, validation and semantics sit together. This is the closest Virgil gets today to per-case methods: a class per instruction instead of a case per instruction.
The core `Opcode` enum gains exactly one case, `EXT`, meaning "look in the extension table". `OpcodePage` gains a parallel `Array<ExtOpcode>`.
A FIXED claim may make any unused byte its prefix page: `undo` uses `0xF9` and `ministe` uses `0xFA`, which is STE's prefix in its binary format. FLOATING claims take free one-byte slots.

An extension shares behavior between its instructions through an ordinary base class (`UndoOp`, `TxnOp`). Virgil has no `super.method()` call, so a subclass that adds to a shared check calls a named helper (`inTransaction(v)`) rather than the overridden method.

### 2. An extension is a name and a list of instructions

```
class InstrExtension(name: string, help: string) {
    def opcodes() -> Array<ExtOpcode>;    // called once, when opcodes are placed
}
```

Hooks that need target-specific types (`emitFastInt`, `emitSpc`) are not built: `InstrExtension` and `ExtOpcode` must compile for every target, and the v3i build does not compile `src/engine/x86-64` (open question 9).

### 3. Registration is one explicit list

Without `--ext`, the build compiles `src/ext/Registered.v3`, whose list is empty:

```
component RegisteredExtensions {
    def all: Array<InstrExtension> = Array<InstrExtension>.new(0);
}
```

With `--ext=undo,...`, `build.sh` generates the list in its place, as `bin/Extensions-<program>.<target>.v3`, from a one-line `src/ext/<name>/extension.txt` per extension (for `undo`, `UndoExtension.new()`):

```
// Generated by build.sh from --ext=undo. Do not edit.
component RegisteredExtensions {
	def all: Array<InstrExtension> = [
		UndoExtension.new()
	];
}
```

A checked-in list naming `UndoExtension` would not compile in a build without `--ext=undo`, so the list follows the flag. No extension file wires itself in, and the generated file shows exactly what is in the build.
With no `--ext`, whole-program compilation removes all extension code.

### 4. Opcode assignment

Run during component initialization, so at build time, before `Opcodes` builds its pages and before fast-int generates its dispatch tables:

1. Core `Opcode` cases are placed, as now.
2. `FIXED` extension opcodes are placed. A collision with core or another fixed opcode is an error naming both. (Today `OpcodePage.put` silently ignores a collision.)
3. `FLOATING` opcodes take leftover slots in list order. They never appear in input modules; the engine writes them in place, as probes do.

### 5. Tier behavior

| Tier | Extension provides | Otherwise |
|---|---|---|
| Validator | `validate` | required |
| v3-int | `interpret` with `ExtFrame` | required |
| fast-int | `emitFastInt` handler in the dispatch table | generic handler calls `runtime_EXT(stack, func, pc)`, which runs `interpret` (modeled on `runtime_PROBE_instr`) |
| SPC | `emitSpc` | existing `bailout`: in the `jit`, `lazy` and `dyn` strategies the function falls back to the interpreter; the `spc` strategy (no fallback) rejects the module |

As built in step 3, fast-int binds one small handler per placed extension opcode (`genExtHandlers`, using `patchDispatchTableAt(prefix, code, pos)`, a variant of `patchDispatchTable` that takes an encoding instead of an `Opcode`). Each handler saves the interpreter state, calls `runtime_EXT`, restores it and dispatches the next instruction. With no extensions, no handler is generated. `emitFastInt` and `emitSpc` are not built yet.

`ExtFrame` is `FrameAccessor` plus `pop`, `push` and an immediate reader.
An extension can ship with only `validate` and `interpret` and behave correctly in every tier.

### 6. Visitors

`BytecodeVisitor` gains one method:

```
def visitExt(op: ExtOpcode) { visitOp(Opcode.EXT); }
```

`BytecodeIterator.dispatch0` handles `Opcode.EXT` by calling `op.owner.accept(op, v)`.

- Visitors that need only stack effects (`ControlStack`, `CfgPrinter`, most monitors) read `op.sig` and need no extension code.
- Visitors that care about a specific instruction override `visitExt`.
- An extension that must change a particular visitor overrides `accept` and type-tests the visitor (double dispatch on the extension side).

Structured instructions would declare themselves `BLOCK_LIKE` with a block type immediate and reuse the core `block`/`end` handling in the validator, `Sidetable` and `ControlStack`. This is not built yet; see the proof of concept and open question 8.

### 7. State without new fields

- Per-type state hangs off the existing `infos` chains.
- `canon_id` serves as a global type id where one is needed.
- Per-instance state uses one slot on `Instance`, declared in core and typed as an open chain, so extensions attach state without adding fields.
- These become weak maps when Virgil has them.

### 8. Memory backends

Transactions change how every store behaves, which instruction hooks alone cannot express.
`Target.newMemory` is a single `def` today (`NativeWasmMemory.new` on x86-64, `V3Memory.new` on v3).
It becomes a per-memory choice: the default for ordinary memories, or an extension's `Memory` subclass for a memory the host marks (initially by a command-line option, with the module importing that memory).
This mirrors the existing `--object-model` choice.

**Write intent.** A backend must be able to tell a store from a load, and today it cannot. In v3-int, loads and stores both call `decodeMemArgAndGetMemoryRange`, which calls `memory.range_oil_64(offset, index, size)`. The interpreter then reads or writes the returned `Range<byte>` directly (`doLoad`, `doStore`). So the seam needs one more core piece: a write-intent call that every write path uses. It could be a `rangeForWrite(offset, index, size)` returning the range, or a `beforeWrite(offset, size)` notification. The default implementation is the existing range lookup, so ordinary memories pay nothing. The write paths that must use it:

- `doStore` and `doStoreLane`;
- the bulk operations: `memory.fill`, `memory.copy` and `memory.init`;
- atomic read-modify-write and store operations;
- host functions that write guest memory, such as WASI's `fd_read`.

**As built in step 2: a write observer.** `Memory` gains `write_range_oil_64` and `write_range_ol_64`, which return the same range as `range_oil_64` and `range_ol_64` but first call `writeObserver.beforeWrite(memory, pos, size)` if an observer is installed. The v3 interpreter's store paths use them (`doStore`, `doStoreLane`, the atomic store, read-modify-write, exchange and compare-exchange helpers), and so do `Memory.fill`, `copyM`, `copyIn` and `readIn`, which covers `memory.fill`, `memory.copy`, `memory.init` and host reads into memory through `readIn`. With no observer the cost is one null check per write. An observer is enough for `undo`, which only needs to see writes; a durable backend that must change where bytes live (a RAM working copy over persistent storage) still needs the per-memory choice above. Host functions that write guest memory directly through `range_*`, such as some of WASI, are not covered yet.

**Fast tiers.** fast-int and SPC keep memory 0's base in a register (`mem0_base`) and generate inline loads and stores that never call into `Memory`. A backend that must observe stores therefore needs one of two things. One is write-protect tracking: the first store to a page faults and marks it dirty, so codegen does not change. The other is a per-memory flag at the bounds check that sends stores to a slow path through the write-intent call.

**As built in step 3: write-protect tracking.** It costs nothing when no observer is installed and changes no generated code. While an observer is installed on a `NativeWasmMemory`:

- the memory's pages are made read-only (`Mmap.protect`, 4 KiB OS pages);
- the first store to a page from fast-int or SPC code faults. Both signal handlers (`X86_64InterpreterCode.handleSignal` and `X86_64SpcModuleCode.handleSignal`) first ask `NativeMemoryTracking.handleWriteFault(addr)`. For a tracked page, it copies the page into a buffer allocated when tracking started (no allocation in the signal handler), makes the page writable and returns true, and the store is retried;
- writes through `write_range_*` (the runtime's bulk operations) save and open their pages directly, so Virgil code never faults;
- `flushWriteTracking()` later reports the saved pages to the observer as `savedImage(memory, pos, image)`, outside the signal handler, and `restartWriteTracking()` makes every page read-only again so the next first write to each page is reported.

Not covered: pages added by `memory.grow` while tracking, memories whose size is not a multiple of 4 KiB (custom page sizes), and host functions that write guest memory without `write_range_*`.

### 9. Block-shaped instructions

An instruction whose `opensBlock()` is true takes a block type as its first immediate. The validator opens an ordinary block for it, closed by an ordinary `end`, and remembers the instruction in the block's control entry (`ControlEntry.ext`).

- **Code at the end.** When the validator reaches that `end`, it calls `onEnd(v, pos)`. The instruction may replace the `end` byte with a one-byte extension opcode (`v.rewriteOpcode(pos, op)`). `ministe` replaces it with `tend`, a FLOATING claim that commits. Branches to the block's own label land on that byte, so they commit too.
- **Validating twice.** Wizard validates a function more than once, so an opcode that replaced an `end` declares `endsBlock()`, and the validator treats it as `end` the next time.
- **Leaving early.** `br`, `br_if`, `br_table` and `return` that leave such a block call `onBranchOut(v)`; `ministe` rejects them. Other ways out (`br_on_*`, `return_call*`, `throw`, `delegate`) are not checked yet.
- **Nesting rules.** An instruction can require an enclosing block of its extension with `v.enclosingExt(owner)`.

Rewriting bytecode in the validator is also how the reference implementation handles transaction ends and branches out of transactions.

## Proof of concept: `undo`

Built with `--ext=undo` (`src/ext/undo/Undo.v3`, tests in `test/ext/undo/`). A volatile, in-memory miniature of a transaction:

```
undo.begin     0xF9 0x00   [] -> []     open a checkpoint
undo.commit    0xF9 0x01   [] -> []     keep the writes since the innermost checkpoint, close it
undo.rollback  0xF9 0x02   [] -> []     restore every byte written since the innermost checkpoint, close it
undo.depth     0xF9 0x03   [] -> [i32]  the number of open checkpoints
```

Committing a nested checkpoint folds its writes into the enclosing one. `commit` and `rollback` with no open checkpoint trap. The instructions act on memory 0, so the extension's `validate` hook rejects them in a module without a memory.

| Part | In `undo` |
|---|---|
| Fixed claims on its own prefix page | `0xF9 0x00..0x03` (moved from `0xFA`, which STE uses) |
| Signature checked by the core validator | all four; `undo.depth` pushes an `i32` |
| `validate` hook | the module must have a memory |
| `interpret` hook with `ExtFrame` | all four |
| `visitExt` and `accept` | the default `accept`, so every visitor sees `visitExt` |
| Per-instance state | `UndoLog` in the `Instance.ext_state` chain |
| Memory write observer | `UndoObserver` records the before-image of every write, byte-exact, so mixed-width and unaligned stores and bulk operations roll back correctly |
| Tier fallbacks | v3-int; fast-int via `runtime_EXT`; SPC bailout, so a function using `undo` stays in the interpreter while its callees are compiled |
| Build-time opt-in | without `--ext=undo`, `0xF9 0x00` fails validation as an invalid opcode |

**Why not `undo.block ... end`.** The first version of this design used a structured `undo.block` whose matching `end` would commit. Doing that needs two things the core does not have: a way for an extension to run code at a particular `end` (the interpreter executes `end` without knowing which block it closes), and a rule for branches that leave the block without reaching its `end`. Both are real design questions for transactions, so `BLOCK_LIKE` is deferred to its own step, and `undo` uses unstructured `begin`/`commit`/`rollback`.

Durable transactions later replace the in-memory undo log with a redo log on persistent storage.

## Second extension: `ministe`

Built with `--ext=ministe` (`src/ext/ministe/MiniSte.v3`, tests in `test/ext/ministe/`). A single-threaded subset of the Wasm Simple Transactions proposal, with STE's encodings:

```
tblock <blocktype> ... end  0xFA 0x02   run the body as a transaction; it commits at its end
tfail                       0xFA 0x0F   [i32] -> []: roll back every open transaction, then trap
tglobal.get <global>        0xFA 0x23   [] -> [t]
tglobal.set <global>        0xFA 0x24   [t] -> []        undone if the transaction fails
i32.tload <memarg>          0xFA 0x28   [i32] -> [i32]   memory 0
i32.tstore <memarg>         0xFA 0x36   [i32 i32] -> []  memory 0
```

All but `tblock` must be inside a `tblock`. Not in the subset: `tblock`'s else clause, `ttry`, and branches or returns that leave a `tblock`, which are rejected. Without `ttry` nothing can catch a failure, so `tfail` rolls back every open transaction and traps.

It needs what `undo` does not: immediates (block type, memory argument, global index), a block-shaped instruction, code at the block's end, a one-byte FLOATING opcode, nesting rules, and a signature that depends on an immediate (`tglobal.*` have a null `sig` and push and pop in `validate`).

**Tests.**
- 8 execution tests on every tier: commit at the end; `br` to the block's own end; `tload`; `tfail` rolling back memory, nested transactions, a compiled callee's store, and a global; `tglobal` commit.
- 5 validation tests: `tstore` and `tfail` outside a `tblock`, `br` and `return` out of one, and valid nesting.
- With `undo` and the registry tests, all 108 pass on the four x86-64-linux tiers, and 36 on v3i.
- With rollback disabled, exactly the 4 rollback tests fail.

## Steps

Each step is a small PR with a check that shows it is done.

1. **Registry, no behavior change.** `ExtOpcode`, `Opcode.EXT`, the `0xFA` page, parallel page tables, the assignment pass and collision error, an empty `RegisteredExtensions`, and `--ext` in `build.sh`. *Done when* spec and unit tests are identical before and after, on x86-64-linux and v3i.
2. **`undo` on v3-int** (done). The `validate`, `interpret` and `accept` hooks, `ExtFrame`, `visitExt`, the memory write observer and write-intent ranges on every v3-int write path, the `Instance` state chain, the generated registration list, and `test/ext/undo/`. 13 execution tests (nesting, commit and rollback, mixed-width and unaligned stores, `memory.fill`, `memory.copy`, traps) and a validation test pass on v3i with `--ext=undo`; disabling the before-image log makes exactly the 6 rollback tests fail. Without the flag, `0xFA 0x00` is an invalid opcode. Until step 3, the undo tests register only on targets whose only test tier is the v3 interpreter.
3. **`undo` in the fast tiers** (done). `runtime_EXT` and a fast-int handler per extension opcode; SPC bailout with interpreter fallback; write-protect tracking for fast-tier stores; `ExtFrame` pops take a type, since the native value stack needs one. All 16 undo tests pass on all four x86-64-linux test tiers (`int`, `jit`, `dyn`, `lazy`), including stores made by a compiled callee and stores on two OS pages; with the page save disabled, exactly the 6 tests whose stores come from Wasm code fail on every tier. Not built: a hand-written `emitFastInt` handler, which is a speed demonstration and moves to step 4.
4. **Measure and review** (done; see Results). The hand-written `emitFastInt` handler was dropped: it needs a target-specific way to register code, which is open question 9.
5. **Persistence library.** Bring the persistent-region and write-ahead-log code from the `pwregions` work under `src/ext/persist/`, built only with `--ext=persist`.
6. **`PersistentWasmMemory`.** The `undo` design with a durable backend: a RAM working copy (the write-ahead log is redo-only), byte-exact dirty lines, commit through the log, recovery at instantiation.
7. **Wasm Simple Transactions** on the registry, with their real encodings.
8. **After the stable language revision:** extensible enums replace `ExtOpcode`, weak maps replace side tables.

## Results (step 4, 2026-10-20)

**Code.** The branch changes 28 files (+1,094/−29 lines, including this document). The core gained 459 lines in `src/engine`, `src/util` and `src/monitors` (26 removed). That is a one-time cost for the seams: `Opcodes.v3` (131), `NativeMemory.v3` (112, write tracking), `Memory.v3` (44), `InstrExtension.v3` (36), `X86_64Interpreter.v3` (32), `V3Interpreter.v3` (31), and small changes elsewhere. `undo` itself is 279 lines in `src/ext/undo/` and `test/ext/undo/` and needed no further core changes.

**Cost when unused.** Five Virgil benchmarks (`bench/` in the Virgil repo: BubbleSort, Fannkuch, MatrixMult, DeltaBlue, Mandelbrot) were compiled to Wasm/WASI and run on three `wizeng` builds: default, with an empty extension that claims no opcodes, and with `--ext=undo`. None of the benchmarks uses an extension. Each figure is the median of 5 interleaved runs on the large inputs, in an x86-64 Linux container on Apple silicon (Rosetta).

| benchmark | mode | default (s) | empty extension | `--ext=undo` |
|---|---|---|---|---|
| BubbleSort | int | 379.13 | +0.3% | −24.8% |
| BubbleSort | jit | 6.19 | −0.5% | +0.3% |
| Fannkuch | int | 16.17 | +0.4% | −23.8% |
| Fannkuch | jit | 0.51 | −2.0% | −2.0% |
| MatrixMult | int | 19.73 | +0.7% | −24.0% |
| MatrixMult | jit | 0.40 | −2.5% | +0.0% |
| DeltaBlue | int | 135.21 | +0.3% | −26.2% |
| DeltaBlue | jit | 7.21 | +2.4% | −3.3% |
| Mandelbrot | int | 3.95 | −0.3% | −19.0% |
| Mandelbrot | jit | 0.36 | −2.8% | +2.8% |

- An extension compiled in but unused costs nothing measurable: the empty extension is within ±0.7% of default in the interpreter, and all `jit` results are within ±3% (noise at these sizes).
- The `--ext=undo` interpreter is 19–26% *faster* on every benchmark. That cannot come from `undo`, whose code never runs here. The visible difference is that `undo` claims the `0xFA` page, which adds a dispatch table and four handlers and so moves the fast interpreter's code. This is most likely a code-layout effect on indirect dispatch, possibly amplified by Rosetta. It needs measuring on native x86-64 hardware before anything is read into it, and it is a hint that fast-int's handler layout matters more than expected.
- Not measured: the per-store null check the v3 interpreter now makes for the write observer (the v3 target is not a performance target), and the cost of write tracking itself while a checkpoint is open (one page fault and a 4 KiB copy per page first written in each checkpoint).

## Open questions

1. Each fixed-encoding extension claims its own prefix byte (`undo` `0xF9`, `ministe` `0xFA`), created only when the extension is built in. Is claiming new prefix bytes the right interim, or should extension opcodes share existing pages?
2. Is `canon_id` stable and global enough to name a type in an off-heap object?
3. Is generating the registration list from `--ext` and each extension's `extension.txt` acceptable, or would a separate main file per configuration be better?
4. Step 3 chose write-protect tracking for fast-tier stores (no codegen change, no cost when unused), with a hook in both signal handlers ahead of the out-of-bounds trap. Is that acceptable, or would a per-memory slow-path flag at the bounds check fit Wizard better?
5. Step 2 combines both forms: `write_range_*` returns the range and calls the write observer first. Is that acceptable, and is routing every write path through it (stores, bulk operations, atomics, host writes) acceptable?
6. Is one core-declared extension slot on `Instance` acceptable?
7. A placement conflict stops the build, because placement runs during compile-time initialization. On native targets, though, `System.error` cannot run at compile time, so the build reports only `ExceptionInInitialization: !NotFoldable` with a stack through `Opcodes.checkPlaced`. The v3i build prints the real message (for example `demo.clash (extension demo): slot already used by unreachable`). Is there a better way to report a compile-time initialization failure with a message on native targets?
8. How should structured extension instructions (`BLOCK_LIKE`) work: how does an extension run code at the `end` that closes its block, and what happens on a branch that leaves the block without reaching `end`?
9. How should an extension supply target-specific code such as an `emitFastInt` handler? `InstrExtension` must stay target-neutral (the v3i build does not compile `src/engine/x86-64`), so this needs a second, per-target registration list, for example generated by `build.sh` from `src/ext/<name>/x86-64-linux/`.
10. The `--ext=undo` fast interpreter ran 19–26% faster on benchmarks that never use it, while an empty extension changed nothing. Is that a known sensitivity of fast-int to code layout, and is it worth investigating on native hardware?
11. Is `undo` a good proof of concept, or would another client (such as call-optimizing opcodes) demonstrate the system better?
