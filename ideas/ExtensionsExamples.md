# Wizard Extensions: Version 1 and Version 2

Nov 19, 2026 · Tim Richards

## Summary

Both versions work, run the same tests, and run equally fast; they differ in how an extension instruction is written and in what it costs Virgil. **Version 1** uses only today's Virgil: each new instruction is a small class, and the core reaches it through one placeholder opcode, `EXT`. **Version 2** uses open enums: each new instruction is a real case of Wizard's `Opcode` enum, with its own methods.

Version 2 reads like the rest of Wizard, needs about 100 fewer lines in the core, and cannot forget to register an instruction. Version 1 needs no change to Virgil and makes a smaller Wizard binary when extensions are compiled in.

**Recommendation:** build on Version 1 now, and move to Version 2 once open enums are merged into Virgil ([titzer/virgil#843](https://github.com/titzer/virgil/pull/843)). The move is mechanical: the same tests pass unchanged on both, and both extensions were rewritten in an afternoon.

The code is on [timdrichards/wizard-engine](https://github.com/timdrichards/wizard-engine): branches `ext-v1` and `ext-v2`. The measurements were taken on 2026-10-27 in an x86-64 Linux container on a Mac.

## The problem

An instruction set extension adds new Wasm instructions to Wizard without editing the core files for each one. Today every instruction lives in the `Opcode` enum and in large `match` statements spread across the validator, the interpreters and the compilers.

To add one instruction, an extension has to say:

1. **Its encoding:** a prefix byte and a code. Some extensions need an exact encoding because real Wasm files use it (transactions); others can take any free byte, because only the engine writes them.
2. **How to validate it:** its stack signature, its immediates, and any extra rule.
3. **What it does:** its meaning in the v3 interpreter, which every other tier can fall back to.
4. **Its state:** data kept per Wasm instance, such as an undo log.

Ben set the rules on 2026-09-17:

- No fields added to a class from outside it.
- No per-case methods on enums for now.
- Extensible enums wait for a stable Virgil language revision.
- A configuration step at build time, where extensions register what they add.
- Two kinds of opcode claim: fixed and "any free one".

Version 1 follows all five rules. Version 2 breaks the second and third on purpose, to show what open enums would buy.

## Version 1 in brief: a class per instruction

In Version 1, each new instruction is a small class that extends `ExtOpcode` and overrides the hooks it needs: `validate`, `interpret`, and a few for block-shaped instructions. An extension is a class that extends `InstrExtension` and returns the list of its instructions.

The `Opcode` enum cannot grow, so the core adds one case, `Opcode.EXT`. It means "this slot belongs to an extension; look it up". Each opcode page gets a second, parallel table that holds the `ExtOpcode` for every `EXT` slot. To run an instruction, the core reads `EXT` from the normal table, then looks up the real instruction in the parallel table (`CodePtr.ext_opcode_at`).

Extensions are switched on at build time: `build.sh --ext=undo,ministe` compiles `src/ext/<name>/` and writes `src/ext/Registered.v3`, a list built from each extension's `extension.txt`. When `Opcodes` is initialized, it places every instruction: exact encodings first (`FIXED`), then the ones that take any free byte (`FLOATING`). A clash stops the build.

## Version 2 in brief: a case of an open enum

In Version 2, Wizard's `Opcode` enum is open: it ends with `_`, and an extension adds its own cases in `enum Opcode.Undo { ... }` in its own file. `Opcode` declares eight hook methods with default bodies, and each extension case overrides the ones it needs, right next to its encoding.

There is no `EXT` placeholder, no parallel table, no `ExtOpcode` class and no registration list. Extension instructions are real `Opcode` values, so the normal tables, `EnumSet`s and `match` statements work on them unchanged. A core `match` sends anything it does not know to its `_` arm, which calls the case's own method (`opcode.interpret(...)`). An extension is an extension because its tag comes after the last core opcode (`Opcodes.isExt`).

`build.sh --ext=undo` still chooses which folders to compile, but compiling the files is all the registering there is. Placement works the same way as in Version 1. A floating instruction says so with `isFloating()` instead of a `FLOATING` claim.

Version 2 needs open enums with per-case methods in Virgil. That is Eliot's work, now [titzer/virgil#843](https://github.com/titzer/virgil/pull/843), and it is not yet merged.

## Example 1: undo

`undo` adds four instructions on prefix `0xF9`: `undo.begin` opens a checkpoint on memory 0, `undo.commit` keeps the writes, `undo.rollback` puts back every byte written since the checkpoint, and `undo.depth` returns how many checkpoints are open. All four share one rule: the module must have a memory.

Below is the instruction part of `src/ext/undo/Undo.v3` in each version, exactly as it is on the branches. The rest of the file, the undo log and the write observer, is the same in both. The whole file is 138 lines in Version 1 and 125 in Version 2.

**Version 1** (`ext-v1`):

```
class UndoExtension extends InstrExtension {
	new() super("undo", "undo checkpoints for memory 0 (proof of concept)") { }
	def opcodes() -> Array<ExtOpcode> {
		return [UndoBegin.new(), UndoCommit.new(), UndoRollback.new(), UndoDepth.new()];
	}
}

// The undo instructions share a prefix and a validation rule.
class UndoOp extends ExtOpcode {
	new(code: u16, mnemonic: string, sig: SigDecl)
		super(0xF9, code, mnemonic, ImmSigs.NONE, sig, OpcodeClaim.FIXED) { }
	// Every undo instruction acts on memory 0, so the module must have one.
	def validate(v: CodeValidator) -> bool {
		if (v.module.memories.length > 0) return true;
		v.err_atpc().OobIndex("memory", 0, 0);
		return false;
	}
}
class UndoBegin extends UndoOp {
	new() super(0x00, "undo.begin", SigCache.v_v) { }
	def interpret(f: ExtFrame) -> TrapReason { return UndoLogs.of(f.instance()).begin(); }
}
class UndoCommit extends UndoOp {
	new() super(0x01, "undo.commit", SigCache.v_v) { }
	def interpret(f: ExtFrame) -> TrapReason { return UndoLogs.of(f.instance()).commit(); }
}
class UndoRollback extends UndoOp {
	new() super(0x02, "undo.rollback", SigCache.v_v) { }
	def interpret(f: ExtFrame) -> TrapReason { return UndoLogs.of(f.instance()).rollback(); }
}
class UndoDepth extends UndoOp {
	new() super(0x03, "undo.depth", SigCache.v_i) { }
	def interpret(f: ExtFrame) -> TrapReason {
		f.pushi(UndoLogs.of(f.instance()).depth());
		return TrapReason.NONE;
	}
}
```

Plus `src/ext/undo/extension.txt`, one line that the build copies into the registration list:

```
UndoExtension.new()
```

**Version 2** (`ext-v2`):

```
enum Opcode.Undo {
	BEGIN(0xF9, 0x00, "undo.begin", ImmSigs.NONE, SigCache.v_v) {
		def interpret(f: ExtFrame) -> TrapReason { return UndoLogs.of(f.instance()).begin(); }
	},
	COMMIT(0xF9, 0x01, "undo.commit", ImmSigs.NONE, SigCache.v_v) {
		def interpret(f: ExtFrame) -> TrapReason { return UndoLogs.of(f.instance()).commit(); }
	},
	ROLLBACK(0xF9, 0x02, "undo.rollback", ImmSigs.NONE, SigCache.v_v) {
		def interpret(f: ExtFrame) -> TrapReason { return UndoLogs.of(f.instance()).rollback(); }
	},
	DEPTH(0xF9, 0x03, "undo.depth", ImmSigs.NONE, SigCache.v_i) {
		def interpret(f: ExtFrame) -> TrapReason {
			f.pushi(UndoLogs.of(f.instance()).depth());
			return TrapReason.NONE;
		}
	};

	// Every undo instruction acts on memory 0, so the module must have one.
	def validate(v: CodeValidator) -> bool {
		if (v.module.memories.length > 0) return true;
		v.err_atpc().OobIndex("memory", 0, 0);
		return false;
	}
}
```

**What to notice:**

- **The shared rule.** Version 1 puts it in a base class, `UndoOp`; Version 2 puts it on the subtype enum, and every case gets it.
- **The encoding.** Version 1 splits it between the base class (prefix `0xF9`, `FIXED`) and each subclass (the code). Version 2 writes the whole encoding on each case, like the core opcodes in `Opcodes.v3`.
- **The list.** Version 1 lists the instructions twice: as classes, and again in `opcodes()`, plus `extension.txt`. Version 2 has no list: the cases are the list.
- **The meaning** (`interpret`) is the same code in both.

## Example 2: mini transactions (ministe)

`ministe` is a single-threaded subset of Wasm Simple Transactions, using STE's real encodings on prefix `0xFA`. It needs three things `undo` does not: immediates, a block-shaped instruction (`tblock ... end`), and code that runs at the block's end. It has seven instructions: `tblock`, `tend`, `tfail`, `tglobal.get`, `tglobal.set`, `i32.tload` and `i32.tstore`. The file is 225 lines in Version 1 and 204 in Version 2.

The interesting part is `tblock` and its end. When the validator reaches the `end` of a `tblock`, it calls `onEnd`, and `tblock` rewrites that one byte into `tend`, an internal instruction that commits. `tend` is a floating instruction: it takes any free one-byte slot.

**Version 1:**

```
class TBlock extends TxnOp {
	def tend: TEnd;
	new(tend) super(0x02, "tblock", ImmSigs.BLOCKT, null) { }
	def opensBlock() -> bool { return true; }
	def validate(v: CodeValidator) -> bool { return true; } // may appear anywhere
	// The block's end commits: replace it with tend.
	def onEnd(v: CodeValidator, pos: int) {
		v.rewriteOpcode(pos, tend);
	}
	def onBranchOut(v: CodeValidator) -> bool {
		v.err_atpc().setc(WasmError.INVALID_OPCODE, "branch or return out of a tblock (not supported)");
		return false;
	}
	def interpret(f: ExtFrame) -> TrapReason {
		TxnLogs.of(f.instance()).begin();
		return TrapReason.NONE;
	}
}
// Internal: replaces the end of each tblock. One byte, at any free slot.
class TEnd extends ExtOpcode {
	new() super(0, 0, "tend", ImmSigs.NONE, SigCache.v_v, OpcodeClaim.FLOATING) { }
	def endsBlock() -> bool { return true; }
	def interpret(f: ExtFrame) -> TrapReason {
		TxnLogs.of(f.instance()).commit();
		return TrapReason.NONE;
	}
}
```

**Version 2:**

```
enum Opcode.Txn {
	TBLOCK(0xFA, 0x02, "tblock", ImmSigs.BLOCKT, null) {
		def opensBlock() -> bool { return true; }
		def validate(v: CodeValidator) -> bool { return true; } // may appear anywhere
		// The block's end commits: replace it with tend.
		def onEnd(v: CodeValidator, pos: int) { v.rewriteOpcode(pos, Opcode.Txn.TEND); }
		def onBranchOut(v: CodeValidator) -> bool {
			v.err_atpc().setc(WasmError.INVALID_OPCODE, "branch or return out of a tblock (not supported)");
			return false;
		}
		def interpret(f: ExtFrame) -> TrapReason {
			TxnLogs.of(f.instance()).begin();
			return TrapReason.NONE;
		}
	},
	// Internal: replaces the end of each tblock. One byte, at any free slot.
	TEND(0x00, 0x00, "tend", ImmSigs.NONE, SigCache.v_v) {
		def isFloating() -> bool { return true; }
		def endsBlock() -> bool { return true; }
		def validate(v: CodeValidator) -> bool { return true; }
		def interpret(f: ExtFrame) -> TrapReason {
			TxnLogs.of(f.instance()).commit();
			return TrapReason.NONE;
		}
	},
	... // tfail, tglobal.get, tglobal.set, i32.tload, i32.tstore
```

The rule that every instruction but `tblock` must be inside a `tblock` is written once in each version:

| | Version 1 | Version 2 |
| --- | --- | --- |
| Where the shared rule lives | base class `TxnOp`, a second base class `TxnMemOp` for loads and stores | methods on `enum Opcode.Txn`, a helper `memoryZero` for loads and stores |
| How it finds the enclosing `tblock` | `TBlock.?(v.enclosingExt(owner))`: a type test on an object | `v.inBlockOf(Opcode.Txn.TBLOCK)`: compare with an enum value |
| How `tblock` names `tend` | an object passed to its constructor | `Opcode.Txn.TEND` |
| Calling the shared rule from an override | by name (`inTransaction(v)`), since Virgil has no `super.validate()` | the same |

The other five instructions are on the branches in `src/ext/ministe/MiniSte.v3`, and follow the same pattern.

## Where the extensions plug into the core

Each place below is changed once, for all extensions. Adding a new extension or a new instruction touches none of them. Version 1 changes 16 core files (558 lines added); Version 2 changes 14 (450 lines added).

### 1. The v3 interpreter's dispatch

The interpreter's big `match` on the opcode gets one new arm, the same idea in both versions.

Version 1 (`V3Interpreter.v3`):

```
EXT => {
	var ext = codeptr.ext_opcode_at(pc);
	if (ext == null) {
		trap(TrapReason.INVALID_OPCODE);
	} else {
		if (extFrame == null) extFrame = V3ExtFrame.new(this);
		var start = codeptr.pos;
		extFrame.imm.reset(codeptr.data, start, codeptr.limit);
		var reason = ext.interpret(extFrame);
		codeptr.at(start);
		codeptr.skip_immediates(ext.imms);
		if (reason != TrapReason.NONE) trap(reason);
	}
}
```

Version 2:

```
_ => { // an extension opcode
	if (extFrame == null) extFrame = V3ExtFrame.new(this);
	var start = codeptr.pos;
	extFrame.imm.reset(codeptr.data, start, codeptr.limit);
	var reason = opcode.interpret(extFrame);
	codeptr.at(start);
	codeptr.skip_immediates(opcode.imms);
	if (reason != TrapReason.NONE) trap(reason);
}
```

- **Version 1:** two steps. The table says `EXT`, then a second lookup re-reads the bytes and finds the `ExtOpcode`. It needs a check for a slot with no instruction.
- **Version 2:** one step. The opcode already is the instruction, so the arm calls its method directly.
- **Better:** Version 2, slightly. It is shorter and cannot hit an empty slot. Speed is the same (see Measurements).

### 2. The validator

The validator gets three things in both versions: an arm that checks the signature and calls `validate`; support for block-shaped instructions (`opensBlock`, then `onEnd` at the block's `end`); and a check on `br`, `br_if`, `br_table` and `return`, which asks each extension block being left whether that is allowed (`onBranchOut`).

- **Version 1:** a new `EXT` arm, plus a lookup at the top of the loop so that `tend`, which replaced an `end`, is read as `end` when a function is validated again.
- **Version 2:** the same code inside the existing `_` arm, behind `if (Opcodes.isExt(opcode))`. The `tend` check is one line: `if (Opcodes.isExt(opcode) && opcode.endsBlock()) opcode = Opcode.END;`.
- **Better:** Version 2, slightly: no second lookup. The block handling is identical, and is the part of the design most worth reviewing in either version.

### 3. The opcode tables and placement (`Opcodes.v3`)

This is where the two differ most, about 135 lines in each.

- **Version 1:** adds the `EXT` case, a parallel `ExtOpcode` table on every opcode page, `findExt` to read it, and `placeExtensions`, which places each extension's list.
- **Version 2:** adds `_` to make `Opcode` open, the eight hook methods with default bodies, `isExt` (a tag test), and the same placement code, which finds extension cases by iterating over `Opcode`.
- **Pros of Version 1:** the `Opcode` enum and its tables keep their meaning; nothing in the core sees an unknown case.
- **Cons of Version 1:** two tables must agree, and any code that iterates over `Opcode` or keys a table by it does not see extension instructions unless it learns about `EXT`.
- **Pros of Version 2:** extension instructions work with everything that already uses `Opcode`: tables by tag, `EnumSet`s, `match`, debug names.
- **Cons of Version 2:** in a build without extensions, `for (op in Opcode)` still yields the `_` placeholder, whose fields are null, so loops over every opcode must skip it (as Version 1's skip `EXT`). The eight hooks also give `Opcode` a method table of about 595 entries each (see Measurements).

### 4. The fast interpreter (`X86_64Interpreter.v3`, `X86_64Runtime.v3`)

The same in both. At startup, `genExtHandlers` writes one small handler per extension instruction into the dispatch table. Each handler calls `runtime_EXT`, which runs the instruction's `interpret` hook and moves the saved `ip` past its immediates. Only the lookup line differs: `ext_opcode_at(pc)` in Version 1, `read_opcode()` plus `isExt` in Version 2.

- **Pro:** an extension never writes assembly to run in the fast interpreter.
- **Con:** every extension instruction is a call out of the interpreter, so it is slower than a hand-written handler. A faster path needs a way for an extension to add code for one target (open question 9 in `ideas/Extensions.md`).

### 5. The single-pass compiler (JIT)

No change in either version for instructions. The compiler meets an extension instruction like any instruction it cannot compile and bails out: in the `jit`, `lazy` and `dyn` modes that function stays in the interpreter while its callees are compiled, and the `spc` mode, which has no interpreter fallback, rejects the module. Its signal handler gets two lines, the same in both, for the page-protection fault described in item 7.

- **Pro:** nothing to write for the JIT.
- **Con:** a function that uses an extension instruction never runs compiled. Ben noted on Nov 16 that a JIT extension point is the last step, and needs an interface for registers and spilling.

### 6. Bytecode visitors and the iterator

`BytecodeVisitor` gets `visitExt`, and each instruction has an `accept` hook to present itself, so monitors and tools see extension instructions.

- **Version 1:** an `EXT` arm in the iterator looks up the `ExtOpcode` and calls `accept`.
- **Version 2:** the `_` arm calls `opcode.accept(v)`; one line.
- **Same in both:** a visitor that cares about one extension instruction must override `visitExt` and test which one it is.

### 7. Memory writes and per-instance state

The same code in both versions, because it has nothing to do with how instructions are declared.

- **Write observer** (`Memory.v3`, `NativeMemory.v3`): every write path in the v3 interpreter calls `write_range_*`, which tells an installed observer before a store. In the fast tiers, stores are direct machine code, so a tracked page is write-protected and the first store to it faults; the handler saves the page and lets the store go on. With no observer installed, the cost is one null check per write in the v3 interpreter.
- **Per-instance state** (`Instance.v3`): one field, `ext_state`, a chain of `InstanceExtState` objects. An extension finds its own entry by type, so it adds no fields to `Instance`, following Ben's rule.

### 8. Turning extensions on

- **Version 1:** `build.sh --ext=undo` compiles `src/ext/undo/` and builds `src/ext/Registered.v3` from each `extension.txt`. Forgetting an instruction in an extension's `opcodes()` list still compiles; a module that uses it then fails with "unsupported opcode".
- **Version 2:** `build.sh --ext=undo` compiles the folder, and that is all. An instruction that is declared is placed. Forgetting is not possible.
- **Better:** Version 2. One less place to keep in step.

## Measurements

All numbers were measured on 2026-10-27 in an x86-64 Linux container on a Mac (Rosetta), and each version is compared with Wizard master built by the same Virgil compiler. Version 2 was built with the open-enum compiler as it stood then. That compiler has since had eight more bug fixes (in [titzer/virgil#843](https://github.com/titzer/virgil/pull/843)), so Version 2's sizes should be measured again before anyone relies on them.

### Code

| | Version 1 | Version 2 |
| --- | --- | --- |
| Wizard core, lines added / removed | 558 / 29 | 450 / 29 |
| Wizard core files touched | 16 | 14 |
| `undo`, lines | 138 | 125 |
| `ministe`, lines | 225 | 204 |
| Extension tests | 271 lines, the same file in both | the same |
| Virgil compiler, lines added | 0 | about 2,000 (open enums) |

### Writing and changing an extension

Each change was made on both branches, built and tested.

| Task | Version 1 | Version 2 |
| --- | --- | --- |
| Add an instruction (`undo.logged`) | 11 lines, 3 places: helper, class, list entry | 9 lines, 2 places: helper, case |
| Change one instruction's rule | 1 line, 1 place | 1 line, 1 place |
| Add a new extension (`counter`, 2 instructions) | 41 lines, 2 files | 36 lines, 1 file |
| Forget to register an instruction | compiles; a module using it fails with "unsupported opcode" | not possible |

One surprise in Version 2: `enum Opcode.Counter` also declared a global type named `Counter`, which clashed with the extension's own class. That was a compiler bug, since fixed in #843: a subtype enum is now named only through its parent.

### Binary size

Size that `undo` and `ministe` add to `wizeng`, as a percentage of master built with the same compiler:

| Target | Version 1 | Version 2 | Version 2, `-compact-mtable` |
| --- | --- | --- | --- |
| x86-64-linux | +1.44% | +2.41% | +1.20% |
| x86-linux | +1.65% | +2.46% | not measured |
| jvm | +0.97% | +3.25% | not measured |
| wasm-wave | +1.85% | +3.24% | not measured |

With both extensions compiled in, Version 2 adds more than Version 1 on every target, about three times as much on the JVM. The cost is the method tables for `Opcode`'s eight hooks, about 595 entries each. Virgil's `-compact-mtable` option halves it on x86-64, below Version 1, and all extension tests pass with it. Without any extensions, Version 2 adds less than Version 1 on every target: 0.13 to 0.25%, against 0.15 to 0.40%. The open-enum compiler itself makes Virgil's compiler about 2.4% larger on x86-64. The byte counts are in [ExtensionsV1V2.md](ExtensionsV1V2.md).

### Speed

The two versions run at the same speed. Time for 200,000 loop iterations, median of 5 runs, in seconds:

| Loop body | V1, interpreter | V2, interpreter | V1, jit | V2, jit |
| --- | --- | --- | --- | --- |
| `block` + `i32.store` (no extension) | 0.10 | 0.11 | 0.10 | 0.10 |
| `undo.begin` + `i32.store` + `undo.commit` | 4.77 | 4.80 | 4.82 | 4.84 |
| `tblock` + `i32.tstore` + `end` | 3.14 | 3.12 | 3.14 | 3.14 |
| `tblock` + `i32.store` + `end` (page fault path) | 4.87 | 4.84 | 4.87 | 5.08 |

- A short transaction costs about 15 to 24 µs in both versions. The time goes into setting up the transaction (page protection, the shadow buffer, a page copy), not into reaching the extension instruction. That is where to optimize.
- The jit gives nothing here, because functions that use extension instructions stay in the interpreter.
- On five Virgil benchmarks compiled to Wasm, with no extension instructions, all builds are the same with the jit. In the fast interpreter they swing ±20% in no consistent direction; even master built by the two compilers differs by up to 15%. That is code layout under Rosetta, and needs repeating on native x86-64 hardware.
- Building the compiler (Virgil bootstrap) took about 7.4 s with upstream and 7.0 s with open enums, the same within noise. Compiling Wizard took 0.5 to 2.4 s per target in every configuration.

### Tests

- The full Wizard suites give the same results for master, Version 1 and Version 2, on x86-64 Linux in all five modes and on v3i.
- 108 extension tests pass on the four x86-64 test modes and 36 on v3i, in both versions. Breaking the rollback code on purpose fails exactly the rollback tests.

## Pros and cons

| | Version 1: classes | Version 2: open enum |
| --- | --- | --- |
| Needs a Virgil change | No | Yes: open enums with per-case methods, not yet merged |
| Looks like the rest of Wizard | No: a separate class world beside `Opcode` | Yes: an instruction is an `Opcode` case, like the core ones |
| Lines in the core | 558 | 450 |
| Registration | a list and `extension.txt` to keep in step | none |
| Works with `match`, tables by tag, `EnumSet` | only through `EXT` and a lookup | directly |
| Binary size with extensions | smaller | larger, unless method tables are compacted |
| Binary size without extensions | larger | smaller |
| Speed | same | same |
| Risk | low: plain classes | higher: the compiler feature has needed 14 bug fixes so far, several of them wrong code on native targets only |

Eliot's point from Nov 16 is the one that matters most for students: in Version 1, writing an extension is a different kind of coding from writing the core, so they have to learn a second way of doing things. Version 2 removes that gap.

### Not built yet: Version 2 without per-case methods

Eliot asked to see Version 2 both with and without per-case methods, and Ben asked for the smallest mechanism that does the job. The Wizard code for "without" is not written yet. The Virgil side exists: it is Eliot's Strategy A, where a subtype enum can declare and override methods but its cases cannot, plus top-level methods ([titzer/virgil#844](https://github.com/titzer/virgil/pull/844)).

In that style, `undo` would put one method on the subtype enum that matches on the case. This is a sketch, not compiled:

```
enum Opcode.Undo {
	BEGIN(0xF9, 0x00, "undo.begin", ImmSigs.NONE, SigCache.v_v),
	COMMIT(0xF9, 0x01, "undo.commit", ImmSigs.NONE, SigCache.v_v),
	ROLLBACK(0xF9, 0x02, "undo.rollback", ImmSigs.NONE, SigCache.v_v),
	DEPTH(0xF9, 0x03, "undo.depth", ImmSigs.NONE, SigCache.v_i);

	def interpret(f: ExtFrame) -> TrapReason {
		var log = UndoLogs.of(f.instance());
		match (this) {
			BEGIN => return log.begin();
			COMMIT => return log.commit();
			ROLLBACK => return log.rollback();
			DEPTH => { f.pushi(log.depth()); return TrapReason.NONE; }
		}
	}
	def validate(v: CodeValidator) -> bool { ... } // as in Version 2
}
```

- **Pro:** this is how the core already handles opcodes, a `match` per task. The compiler needs no per-case method tables, which is where Version 2's extra binary size comes from.
- **Con:** an instruction's behavior is spread over several methods again, one per hook, instead of sitting next to its encoding.

Next step: build it on a third branch and add its numbers here.

## Recommendation

**Build on Version 1 now, and move to Version 2 when open enums are merged into Virgil.**

- **Version 2 is the better way to write an extension.** An instruction is one enum case with its encoding and behavior together, the core is about 100 lines smaller, there is nothing to register, and extension code reads like core code.
- **But it rests on a compiler feature that is not ready.** It has needed 14 compiler fixes so far, six found while building Version 2 and eight in a review, and the structural questions in #843 (syntax, how subtype enums relate to classes, method table layout) are still open. With default options it also costs more binary size on every target.
- **Moving later is cheap.** The hooks in the core are the same in both versions, the same test files pass on both, and both extensions were converted in an afternoon.

Version 1 could go upstream as a first PR once Ben is happy with the hook points above. It would first need a rebase onto current Wizard master.

### Questions for Ben and Eliot

1. Is Version 1 now and Version 2 later the right plan, or would you rather wait for open enums and start with Version 2?
2. With per-case methods or without? The "without" version above trades a `match` per hook for smaller method tables.
3. If Version 2 keeps per-case methods, should Wizard build with `-compact-mtable`, or should the compiler choose the table layout itself?
4. Block-shaped instructions rewrite one `end` byte (`tend`). Eliot suggested one shared "transaction special" opcode plus side-table entries instead. Should extensions be able to add side-table entries?
5. When should an extension get a hook for the JIT, and for hand-written fast-interpreter handlers?
