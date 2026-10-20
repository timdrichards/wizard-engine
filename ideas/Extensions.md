# Wizard Extensions

A proposal for letting an instruction set extension live outside `src/engine/`: it is switched on at build time, registers its opcodes and behavior through a small set of hooks, and degrades gracefully in tiers that have no fast path for it.
Transactions are the motivating client.
A small proof-of-concept extension, `undo`, comes first, to show the mechanism end to end.

Status: draft, for discussion. Branch `extension-registry`.

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

### 1. Extension instructions are descriptor objects

```
class ExtOpcode(prefix: u8, code: u16, mnemonic: string,
                imms: Array<ImmKind>, sig: SigDecl,
                claim: OpcodeClaim, owner: InstrExtension) { }

enum OpcodeClaim { FIXED, FLOATING }
```

These are the same fields as `Opcode`, plus a claim and an owner.
The core `Opcode` enum gains exactly one case, `EXT`, meaning "look in the extension table".
`OpcodePage` gains a parallel `Array<ExtOpcode>`.
Extensions get their own prefix page, `0xFA`, which is unused in Wizard today.
After the stable language revision, `ExtOpcode` and `Opcode.EXT` become extensible enum cases and the rest of this design is unchanged.

### 2. An extension is a class with hooks

```
class InstrExtension(name: string, help: string) {
    def opcodes() -> Array<ExtOpcode>;                          // required
    def validate(v: CodeValidator, op: ExtOpcode) -> bool;      // required
    def interpret(op: ExtOpcode, f: ExtFrame) -> Throwable;     // required: reference semantics
    def emitFastInt(op: ExtOpcode, masm: X86_64MacroAssembler) -> bool { return false; }
    def emitSpc(op: ExtOpcode, spc: SinglePassCompiler) -> bool { return false; }
    def accept(op: ExtOpcode, v: BytecodeVisitor) { v.visitExt(op); }
    def extendTypes(...) { }                                    // placeholder: shape to be decided
    def targets() -> Array<string>;                             // e.g. native-only extensions
}
```

Each hook matches over the extension's own opcodes, so no per-case methods are needed.

### 3. Registration is one explicit list

```
// src/ext/Registered.v3, compiled only with build.sh --ext=...
component RegisteredExtensions {
    def all: Array<InstrExtension> = [UndoExtension.new()];
}
```

No extension file wires itself in; reading this list tells you exactly what is in the build.
With no `--ext`, the list is empty and whole-program compilation removes all extension code.

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
| SPC | `emitSpc` | existing `bailout`; the function stays in the interpreter (to verify) |

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

Structured instructions declare themselves `BLOCK_LIKE` with a block type immediate and reuse the core `block`/`end` handling in the validator, `Sidetable` and `ControlStack`.

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

**Fast tiers.** fast-int and SPC keep memory 0's base in a register (`mem0_base`) and generate inline loads and stores that never call into `Memory`. A backend that must observe stores therefore needs one of two things. One is write-protect tracking: the first store to a page faults and marks it dirty, so codegen does not change. The other is a per-memory flag at the bounds check that sends stores to a slow path through the write-intent call.

## Proof of concept: `undo`

Built with `--ext=undo`. A volatile, in-memory miniature of a transaction:

```
undo.block <blocktype> ... end   ;; stores to memory 0 inside the block are logged
undo.rollback                    ;; restore every byte stored since the innermost undo.block, then exit it
undo.depth -> i32                ;; current nesting depth
```

Encoded as `0xFA 0x00` to `0xFA 0x02`.
It uses every part of the design:

| Part | In `undo` |
|---|---|
| Fixed claims on a prefix page | `0xFA 0x00..0x02` |
| `BLOCK_LIKE` control | `undo.block ... end` |
| Validator rules | `undo.rollback` only inside an `undo.block` |
| Plain op needing only `sig` | `undo.depth` |
| `visitExt` and `accept` | disassembly and tracing of `undo.*` |
| Per-instance state | the undo-log stack |
| Memory backend | `UndoWasmMemory` logs before-images, byte-exact, so mixed-width and unaligned stores roll back correctly |
| Tier fallbacks | v3-int semantics; fast-int via `runtime_EXT`; SPC bailout |
| Build-time opt-in | without `--ext=undo`, a module using `0xFA` fails validation with a clear error |

Durable transactions later swap the in-memory undo log for a redo log on persistent storage behind the same memory-backend interface.

## Steps

Each step is a small PR with a check that shows it is done.

1. **Registry, no behavior change.** `ExtOpcode`, `Opcode.EXT`, the `0xFA` page, parallel page tables, the assignment pass and collision error, an empty `RegisteredExtensions`, and `--ext` in `build.sh`. *Done when* spec and unit tests are identical before and after, on x86-64-linux and v3i.
2. **`undo` on v3-int.** Hooks, `BLOCK_LIKE`, `visitExt`, the memory-backend choice, the write-intent call on every v3-int write path, `UndoWasmMemory`, the `Instance` slot. Tests in `test/ext/undo/` (nesting; rollback of mixed-width and unaligned stores; `memory.fill` and `memory.copy`; traps inside a block; validation errors). *Done when* they pass with `--ext=undo` and the same modules are rejected without it.
3. **`undo` in the fast tiers.** `runtime_EXT` fallback, SPC bailout, one hand-written `emitFastInt` handler, and the chosen approach to fast-tier stores. *Done when* results are identical across interpreter, fast-int and JIT modes.
4. **Measure and review.** Lines in `src/ext/undo/` against lines changed in `src/engine/`, and overhead with the extension compiled in but unused.
5. **Persistence library.** Bring the persistent-region and write-ahead-log code from the `pwregions` work under `src/ext/persist/`, built only with `--ext=persist`.
6. **`PersistentWasmMemory`.** The `undo` design with a durable backend: a RAM working copy (the write-ahead log is redo-only), byte-exact dirty lines, commit through the log, recovery at instantiation.
7. **Wasm Simple Transactions** on the registry, with their real encodings.
8. **After the stable language revision:** extensible enums replace `ExtOpcode`, weak maps replace side tables.

## Open questions

1. Is a separate prefix page (`0xFA`) for extensions the right interim, or should extension opcodes share existing pages?
2. Is `canon_id` stable and global enough to name a type in an off-heap object?
3. Should `--ext` be a `build.sh` flag like `--object-model`, or a separate main file per configuration?
4. For memory backends that observe stores: write-protect tracking in the signal handler, or a per-memory slow-path flag at the bounds check?
5. Which form should the write-intent call take: `rangeForWrite` returning the range, or a `beforeWrite` notification before the existing range lookup? And is routing every write path through it (stores, bulk operations, atomics, host writes) acceptable?
6. Is one core-declared extension slot on `Instance` acceptable?
7. Is `undo` a good proof of concept, or would another client (such as call-optimizing opcodes) demonstrate the system better?
