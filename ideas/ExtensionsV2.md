# Wizard Extensions, Version 2: Open Enums

Version 2 of the extension framework (branch `ext-v2`). It solves the same problem as [Extensions.md](Extensions.md), which describes Version 1, but uses a language feature Virgil does not have yet: **open enums with per-case methods**. An extension instruction is then a case of `Opcode` itself.

This document follows the sections of Extensions.md, so the two can be read side by side. Where V2 is the same as V1, it says so and points there. The measured comparison is in [ExtensionsV1V2.md](ExtensionsV1V2.md).

It needs Virgil from timdrichards/virgil branch `open-enums-rebased`: Eliot Moss's titzer/virgil#653, rebased onto Virgil `908c7e2`, with six compiler fixes (listed in ExtensionsV1V2.md).

Status: built and tested, for discussion.

## Constraints

V2 is built to answer one question: what would the design look like once the language allows it? So it sets aside two of the constraints from discussion with Ben (September 2026), on purpose:

- **Per-case methods on enums.** V2 depends on them.
- **Extensible enums wait for a stable language revision.** V2 uses them now, from an unmerged compiler branch.

It keeps the others:

- **No fields added to classes from outside.** Per-instance state uses the same `Instance.ext_state` chain as V1.
- **A configuration phase at build time.** Opcode placement still runs in component initializers, so at compile time.
- **Fixed and floating claims.** Both are supported.
- **Visitor extensibility.** It gets the same answer as V1, with the case itself as the argument.

## The language feature: open enums

Virgil enums today are closed: every case is listed in the declaration. #653 adds two things.

**An open enum** ends its case list with `_`. Other files may then declare subtype enums that add cases:

```
enum Opcode(prefix: u8, code: u16, mnemonic: string, imms: Array<ImmKind>, sig: SigDecl) {
	INVALID(...), UNREACHABLE(...), ..., SWITCH(...),
	_;
	def validate(v: CodeValidator) -> bool { return true; }
	...
}

// In another file:
enum Opcode.Undo {
	BEGIN(0xF9, 0x00, "undo.begin", ImmSigs.NONE, SigCache.v_v) { ... },
	...
}
```

`Opcode.Undo.BEGIN` is an `Opcode`. It has a tag after every core case, it is listed by `for (op in Opcode)`, and it counts in `Opcode.count`.

**Per-case methods.** An enum may declare methods, and each case may override them in a body after its arguments. A subtype enum may also override them for all of its cases. A case's override wins over its subtype's, which wins over the root's.

Two consequences matter for Wizard:

- Whole-program compilation still sees every case. A build without `--ext` has only the core cases plus the `_` placeholder.
- A `match` on `Opcode` cannot name an extension's cases, since the core does not know them. They fall to the `_` arm.

## Design

### 1. Extension instructions are cases of `Opcode`

`Opcode` gains the `_` and eight hook methods, with defaults. The core never calls them for its own opcodes.

```
enum Opcode(...) {
	..., SWITCH(0x00, 0xE6, "switch", imm.CONT_TAG, null),
	_;
	def validate(v: CodeValidator) -> bool { return true; }           // beyond the signature
	def interpret(f: ExtFrame) -> TrapReason { return TrapReason.UNIMPLEMENTED; }
	def accept(v: BytecodeVisitor) { v.visitExt(this); }
	def isFloating() -> bool { return false; }                         // internal one-byte opcode
	def opensBlock() -> bool { return false; }                         // block-shaped (section 9)
	def onEnd(v: CodeValidator, pos: int) { }
	def onBranchOut(v: CodeValidator) -> bool { return true; }
	def endsBlock() -> bool { return false; }
}
```

These are V1's `ExtOpcode` hooks, moved onto `Opcode`. The rest of V1's machinery goes away:

| V1 | V2 |
|---|---|
| `class ExtOpcode`, one subclass per instruction | one case per instruction |
| `Opcode.EXT`, "look in the extension table" | not needed: the case is in the table |
| a parallel `Array<ExtOpcode>` in each `OpcodePage` | not needed |
| `OpcodeClaim.FIXED` / `FLOATING` | fixed by default; a floating case overrides `isFloating()` |
| `owner`, set when placed | not needed: the case's subtype is its extension |

An extension shares behavior between its instructions with methods on its subtype enum. `ministe` checks "must be inside a `tblock`" once, in `Opcode.Txn.validate`. Cases that need more override it and call the subtype's helpers. Virgil has no `super.method()` in either version, so a case that adds to a shared check calls a named helper (`inTransaction(v)`), as in V1.

### 2. An extension is a subtype enum

There is no `InstrExtension` class. An extension is the file `src/ext/<name>/*.v3`, which declares `enum Opcode.<Name>` and whatever support classes it needs.

Target-specific hooks (`emitFastInt`, `emitSpc`) are still not built. The reason is the same as in V1: `Opcode` must compile for every target, and the v3i build does not compile `src/engine/x86-64` (open question 9 in Extensions.md).

### 3. Registration is compiling the files

`build.sh --ext=undo,ministe` adds `src/ext/<name>/*.v3` to the engine and `test/ext/<name>/*.v3` to the unit tests. An unknown name lists the extensions that exist.

There is no registration list, no generated file and no `extension.txt`. Compiling an extension's files adds its cases to `Opcode`, and that is all it takes. So an instruction cannot be left out of a list by mistake. In V1, that mistake compiles, and a module that uses the instruction then fails validation with "unsupported opcode".

What is in a build is visible in the `--ext` flag, not in a file.

### 4. Opcode assignment

`Opcodes`' component initializer places the core opcodes as before, skipping extension cases, then calls `placeExtensions()`:

1. It collects every extension case: `Opcodes.isExt(op)` is `op.tag > Opcode.SWITCH.tag`. It skips the `_` placeholder, which has no mnemonic.
2. **Fixed** cases go at exactly their prefix and code. As in V1, an unused byte can become an extension's prefix page (`undo` `0xF9`, `ministe` `0xFA`). A collision with a core opcode, an internal opcode or another extension stops the build with an error naming both.
3. **Floating** cases take the first free one-byte slot. Their declared code is ignored, and the placed code is kept in `Opcodes.floating_code[tag]`. `Opcodes.codeOf(op)` returns the code an opcode really has.
4. The list is kept as `Opcodes.ext_opcodes`, for tiers that bind a handler to each encoding.

This is V1's algorithm, run over enum cases instead of `ExtOpcode` objects. `OpcodePage.placeAt(op, code)` puts the case itself in the page, so `Opcodes.find` and the decoder return it with no second table.

### 5. Tier behavior

The same as V1 (Extensions.md, section 5). Each tier reaches the hooks differently.

| Tier | How it reaches an extension opcode |
|---|---|
| Validator | the `_` arm: `isExt(op)`, then the signature check and `op.validate(this)` |
| v3-int | the `_` arm: `op.interpret(extFrame)` |
| fast-int | one handler per case in `Opcodes.ext_opcodes` (`genExtHandlers`), which calls `runtime_EXT(stack, func, pc)`. That reads the opcode at `pc` and calls `op.interpret` |
| SPC | `BytecodeIterator` calls `op.accept(v)`. The default reaches `visitOp`, which bails out, and the function falls back to the interpreter |

`ExtFrame` is unchanged from V1: `imm` (a `CodePtr` at the immediates), `instance()`, typed `pop(t)`, `push`, `popi` and `pushi`. `V3ExtFrame` and `NativeExtFrame` provide it in the two interpreters.

### 6. Visitors

`BytecodeVisitor` gains:

```
def visitExt(op: Opcode) { visitOp(op); }
```

`BytecodeIterator.dispatch0` gains one arm, `_ => opcode.accept(v)`. The three cases from V1 still apply:

- visitors that need only stack effects read `op.sig`;
- visitors that care about one instruction override `visitExt` and compare `op`;
- an extension that must change a particular visitor overrides `accept`.

The difference from V1 is that `visitExt` receives the `Opcode`, so the visitor can compare it with `Opcode.Txn.TBLOCK` directly.

Two core loops that walk every opcode now skip extension cases: `OpcodeMap` and the Whamm monitor's pattern table.

### 7. State without new fields

The same as V1: per-instance state in the `Instance.ext_state` chain (`UndoLog` and `TxnLog` subclass `InstanceExtState`).

### 8. Memory backends

The same as V1, unchanged:

- the write observer (`Memory.writeObserver`, `write_range_oil_64` and `write_range_ol_64`) on every v3-int write path;
- page-protection write tracking in `NativeWasmMemory`, with the hook in both signal handlers.

### 9. Block-shaped instructions

The same rules as V1. The validator's state holds a case instead of a descriptor:

- `ControlEntry.ext` is an `Opcode`, and `INVALID` means "not an extension block".
- `v.inBlockOf(op)` tells whether the code is inside a block opened by `op`. It replaces V1's `enclosingExt(owner)`.
- `v.rewriteOpcode(pos, op)` writes `Opcodes.codeOf(op)`, so a floating case such as `tend` is written at its placed byte.
- Before the main `match`, an extension opcode with `endsBlock()` is read as `END`, so revalidation works.

## Proof of concept: `undo`

Built with `--ext=undo` (`src/ext/undo/Undo.v3`, 125 lines; tests in `test/ext/undo/`, the same files as V1). The instructions and behavior are those in Extensions.md. The whole instruction set is one enum:

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

`validate` is declared once for the subtype, and every case uses it. `UndoLog` and `UndoObserver` are unchanged from V1.

## Second extension: `ministe`

Built with `--ext=ministe` (`src/ext/ministe/MiniSte.v3`, 204 lines; tests in `test/ext/ministe/`, the same files as V1). The subset and encodings are those in Extensions.md. Its block handling shows the hooks working together:

```
enum Opcode.Txn {
	TBLOCK(0xFA, 0x02, "tblock", ImmSigs.BLOCKT, null) {
		def opensBlock() -> bool { return true; }
		def validate(v: CodeValidator) -> bool { return true; } // may appear anywhere
		// The block's end commits: replace it with tend.
		def onEnd(v: CodeValidator, pos: int) { v.rewriteOpcode(pos, Opcode.Txn.TEND); }
		def onBranchOut(v: CodeValidator) -> bool { ...; return false; }
		def interpret(f: ExtFrame) -> TrapReason { TxnLogs.of(f.instance()).begin(); return TrapReason.NONE; }
	},
	// Internal: replaces the end of each tblock. One byte, at any free slot.
	TEND(0x00, 0x00, "tend", ImmSigs.NONE, SigCache.v_v) {
		def isFloating() -> bool { return true; }
		def endsBlock() -> bool { return true; }
		def validate(v: CodeValidator) -> bool { return true; }
		def interpret(f: ExtFrame) -> TrapReason { TxnLogs.of(f.instance()).commit(); return TrapReason.NONE; }
	},
	TFAIL(...), TGLOBAL_GET(...), TGLOBAL_SET(...), I32_TLOAD(...), I32_TSTORE(...);

	// Every transaction instruction but tblock must be inside a tblock.
	def validate(v: CodeValidator) -> bool { return inTransaction(v); }
	def inTransaction(v: CodeValidator) -> bool {
		if (v.inBlockOf(Opcode.Txn.TBLOCK)) return true;
		...
	}
}
```

**Tests.** With `undo` and the registry unit tests, all 108 pass on the four x86-64-linux tiers, and 36 on v3i. These are the same counts and the same extension test files as V1.

## What V2 changes in the core

450 lines added and 29 removed in `src` outside `src/ext`, in 14 files (V1: 558 and 29, in 16 files).

| File | Change |
|---|---|
| `Opcodes.v3` | `_` and the eight hooks on `Opcode`; `isExt`, `codeOf`, `placeExtensions`, `placeFixed`, `placeFloating`, `slotUser`, extension prefix pages; `OpcodePage.placeAt`; `findByName` skips the placeholder |
| `CodeValidator.v3` | the `_` arm for extension opcodes; `ControlEntry.ext`; `checkBranchOut`, `inBlockOf`, `rewriteOpcode`; `endsBlock` read as `END` |
| `ExtFrame.v3` | new: the frame an `interpret` hook sees |
| `V3Interpreter.v3` | the `_` arm; `V3ExtFrame`; stores through `write_range_*` |
| `X86_64Interpreter.v3`, `X86_64Runtime.v3` | `genExtHandlers`, `patchDispatchTableAt`, `runtime_EXT`, `NativeExtFrame`; the write-fault hook |
| `X86_64SinglePassCompiler.v3` | the write-fault hook in its signal handler |
| `BytecodeIterator.v3`, `BytecodeVisitor.v3` | the `_` arm and `visitExt` |
| `Memory.v3`, `NativeMemory.v3`, `Instance.v3` | the write observer, page tracking, `ext_state` (the same as V1) |
| `OpcodeMap.v3`, `WhammMonitor.v3` | skip extension cases |
| `build.sh` | `--ext` compiles `src/ext/<name>/` and `test/ext/<name>/` |

## Things to know when writing a V2 extension

- **Subtype enum names are global types.** `enum Opcode.Counter` also declares a type named `Counter`, so it clashes with any other `Counter` in the program. Pick a name no other type uses.
- **The core's `match` statements cannot name your cases.** Every core switch over `Opcode` reaches extension cases through its `_` arm, which calls a hook. To change how a core tier handles an instruction, add a hook.
- **Loops over every opcode see your cases and the placeholder.** Core code that walks `Opcode` and indexes tables by `op.code` must skip `Opcodes.isExt(op)`. In a build without extensions, the only such case is the `_` placeholder, whose fields are null.
- **A floating case's declared code is not its code.** Use `Opcodes.codeOf(op)`.
- **Initialization order.** `Opcodes` iterates `Opcode` during its own initialization, so every extension's cases must be initialized first. The rebased compiler guarantees this, by initializing the whole enum hierarchy before a field is read. #653 as submitted does not, and every extension case then reads as null.

## Code size

Per-case methods give `Opcode`'s eight hooks method tables indexed by tag, across about 595 opcodes. With both extensions, V2's x86-64 `wizeng` is 2.41% larger than master (V1: 1.44%). The compiler's `-compact-mtable=4` option brings that down to 1.20%, and all extension tests pass with it. Without extensions, V2 is 0.25% larger than master (V1: 0.36%). Full numbers are in ExtensionsV1V2.md.

## Open questions

The open questions in Extensions.md apply here too, except question 3 (the registration list), which V2 removes. V2 adds three:

1. Is #653, with the six fixes on `open-enums-rebased`, the form open enums should take in Virgil? In particular, is initializing the whole hierarchy on a field read the right rule for initialization order?
2. Should subtype enum names be scoped under their parent (`Opcode.Counter`) instead of being global types?
3. Should Wizard build with `-compact-mtable` once `Opcode` has per-case methods?
