# Rux Compiler — Bottlenecks & Stability Blockers

> Architectural review of Rux **v0.2.3** on branch `analysis/bottlenecks`.
> Scope: identify the fundamental issues that must be fixed first to make Rux a
> **stable, usable, maintainable** language. Brutally honest, file-and-line specific.

---

## TL;DR — The 7 Things Blocking Stability

| # | Blocker | Where | Why it matters |
|---|---------|-------|----------------|
| 1 | ~~Naive "spill everything" register allocator~~ ✅ Completed ([PR #63](https://github.com/rux-lang/Rux/pull/63)) | [Source/Asm.cpp](Source/Asm.cpp) | Every program runs 10–100× slower than necessary; bloated `.text`. |
| 2 | Zero IR optimization passes | [Source/Lir.cpp](Source/Lir.cpp), [Source/Asm.cpp](Source/Asm.cpp) | No DCE, no constant folding, no inlining — no pass manager exists. |
| 3 | No standard library | (missing) | Users must hand-write `extern` to OS APIs to do anything. |
| 4 | No memory-safety story | [Include/Rux/Type.h](Include/Rux/Type.h), entire backend | Raw pointers, no bounds checks, no lifetimes — UB on day one. |
| 5 | Custom `.rcu` object format + hand-rolled PE/ELF/Mach-O linker | [Source/Rcu.cpp](Source/Rcu.cpp), [Source/Linker.cpp](Source/Linker.cpp) | ~2400 LoC of fragile binary plumbing duplicated per OS. |
| 6 | Test suite is 3 toy programs | [Tests/](Tests/) | No regression safety net; refactors are blind. |
| 7 | Sema is incomplete (no mutability, no exhaustiveness, no generics monomorph.) | [Source/Sema.cpp](Source/Sema.cpp) | Programs that *should* be rejected compile; programs that *should* compile don't. |

Fix these and Rux goes from "demo" to "usable."

---

## Codebase Snapshot

| Component | File | ~LoC | Health | Notes |
|-----------|------|------|--------|-------|
| Lexer | [Source/Lexer.cpp](Source/Lexer.cpp) | 650 | Good | UTF-8 not validated. |
| Parser | [Source/Parser.cpp](Source/Parser.cpp) | 1200 | OK | Weak error recovery. |
| Sema | [Source/Sema.cpp](Source/Sema.cpp) | 2500 | Fair | Monolithic, missing checks. |
| HIR | [Source/Hir.cpp](Source/Hir.cpp) | 400 | Good | — |
| LIR | [Source/Lir.cpp](Source/Lir.cpp) | 300 | Good shape, no opts | No PassManager. |
| ASM | [Source/Asm.cpp](Source/Asm.cpp) | 1100 | **Poor** | Naive regalloc dominates LoC. |
| RCU | [Source/Rcu.cpp](Source/Rcu.cpp) | 900 | Works | Non-standard format. |
| Linker | [Source/Linker.cpp](Source/Linker.cpp) | 1500 | **Risky** | Maintenance trap. |
| CLI / Package | [Source/Cli.cpp](Source/Cli.cpp), [Source/Manifest.cpp](Source/Manifest.cpp), [Source/Package.cpp](Source/Package.cpp) | 2500 | Fair | Many stubbed subcommands. |
| Tests | [Tests/](Tests/) | ~50 | **Critical** | Three programs. |

---

## CRITICAL — Must fix before anything else

### ✅ C1. Naive register allocation (every vreg → unique stack slot) — (Completed · [PR #63](https://github.com/rux-lang/Rux/pull/63))
**Severity:** Critical · **File:** [Source/Asm.cpp](Source/Asm.cpp) (≈L205–L480)

Every LIR virtual register is assigned a unique `[rbp - off]` slot. Every op does
load → compute in `rax`/`xmm0` → store. Only `rax`, `r10`, `r11`, `xmm0`, `xmm1`
are ever used as physical registers.

Sketch from `AllocSlot` / `LoadA` / `StoreA`:

```cpp
int32_t AllocSlot(LirReg reg, int bytes) {
    if (auto it = slotMap.find(reg); it != slotMap.end()) return it->second;
    nextOff = AlignUp(nextOff, al);
    nextOff += (bytes > 0 ? bytes : 8);
    slotMap[reg] = nextOff;     // every vreg gets its own slot, forever
    return nextOff;
}
```

Generated code for `r1 = r0 + 1`:

```asm
mov rax, qword [rbp - 8]      ; load r0
mov r10, qword [rbp - 16]     ; load literal
add rax, r10
mov qword [rbp - 24], rax     ; store r1
```

Optimal would be a single `add` in a callee/caller-saved GPR. This single issue
dominates both code size and runtime performance.

**Fix:** Implement linear-scan or graph-coloring allocator over LIR.
Use the full ABI register set; spill only on pressure. Treat this as the #1
backend work item.

---

### C2. No optimization pass infrastructure
**Severity:** Critical · **Files:** [Source/Lir.cpp](Source/Lir.cpp), [Source/Asm.cpp](Source/Asm.cpp)

There is no `Pass` base class, no `PassManager`, no `--O1/O2`. Consequences:

- No dead code elimination — LIR carries unused defs straight to asm.
- No constant folding at IR level (some at AST, none at LIR).
- No CSE, no LICM, no inlining, no copy propagation.
- No way to add a pass without editing `Asm.cpp` directly.

**Fix:** Introduce `Lir::Pass` + `PassManager`, expose `--dump-lir=<pass>`, then
land DCE → ConstFold → CopyProp → SimplifyCFG in that order. They are cheap and
multiply the value of any future regalloc.

---

### C3. No standard library
**Severity:** Critical · **Files:** none (missing entirely)

Every test program redeclares OS APIs by hand:

```rux
@[Import(lib: "kernel32")]
extern func WriteFile(handle: int, buffer: *char8, count: uint64,
                      written: *uint64, overlapped: int) -> bool;
```

Nothing exists for `Vec`, `String`, `Option`, `Result`, `HashMap`, math,
allocation, formatting, or portable I/O. The `Std::Io::Print` shown in docs is
aspirational.

**Fix:** Define a minimum `Std` package shipped with the compiler:
`Io` (portable print/read), `Mem` (alloc/free), `Option<T>`, `Result<T,E>`,
`Vec<T>`, `Str` methods, `Math`. Write it in Rux to dogfood the language.

---

### C4. No memory-safety model
**Severity:** Critical · **Files:** [Include/Rux/Type.h](Include/Rux/Type.h), [Source/Sema.cpp](Source/Sema.cpp), [Source/Asm.cpp](Source/Asm.cpp)

The language exposes raw `*T` with no lifetime, no bounds checks on `T[]` /
`T[N]` indexing, no null handling, no escape analysis. Programs like
`return *p` after the pointee goes out of scope compile and "work" until they
don't.

**Fix (incremental):**
1. Emit slice bounds checks in codegen (cheap, immediate UB reduction).
2. Add `Option<T>` and a non-nullable reference type `&T` distinct from `*T`.
3. Decide a long-term model: borrow checker (Rust), ARC, or runtime checks
   (Zig-style). Pick **one** and commit; the current "everything is a raw
   pointer" state has no future.

---

### C5. Custom RCU object format + 1500-line bespoke linker
**Severity:** High → Critical for maintenance · **Files:** [Source/Rcu.cpp](Source/Rcu.cpp), [Source/Linker.cpp](Source/Linker.cpp)

`Linker.cpp` hand-builds PE32+, ELF64, and Mach-O headers and writes relocations
by hand. Sample from the PE path:

```cpp
textPre.insert(textPre.end(), {0x48, 0x83, 0xEC, 0x28}); // sub rsp, 0x28
const size_t kCallMainDisp = textPre.size() + 1;
textPre.insert(textPre.end(), {0xE8, 0x00, 0x00, 0x00, 0x00}); // call Main
```

Every new relocation type, every new platform, every debug-info effort
multiplies across three hand-coded backends. There is no DWARF / PDB, no
support for system libraries beyond a custom import-thunk scheme.

**Fix:** Emit **standard ELF/COFF/Mach-O object files** and delegate linking to
`lld` (`clang -fuse-ld=lld` or invoking `ld.lld` / `ld64.lld` / `lld-link`
directly). Keep `.rcu` only as an internal IPC format if useful. This deletes
~1000 LoC of brittle code and unlocks debug info, LTO, and system libraries
for free.

---

### C6. Test suite is three toy programs
**Severity:** Critical · **Path:** [Tests/](Tests/)

Today: Echo, Io, Pow — together ~50 lines. Nothing tests structs, enums,
generics, methods, modules, imports, pattern matching, loops, scoping rules,
overload resolution, error messages, or codegen edge cases. Any non-trivial
change to Sema or Asm could silently break the language and no one would know.

**Fix:**
- Convert tests into a CTest-driven matrix (`enable_testing()` + `add_test`).
- Add ≥1 test per language feature plus golden-file tests for `--dump-ast`,
  `--dump-lir`, `--dump-asm`.
- Add a GitHub Actions matrix (mac / linux / windows).
- Land a small fuzzer over the lexer/parser before adding new syntax.

---

### C7. Semantic analysis gaps
**Severity:** High · **File:** [Source/Sema.cpp](Source/Sema.cpp) (2500 LoC monolith)

Missing or incomplete:

- **Mutability enforcement** — `let x = 5; x = 10;` is not rejected.
- **Match exhaustiveness** — no diagnostic for missing patterns.
- **Generic monomorphization** — generics resolve nominally but aren't
  specialized; codegen for `Vec<T>` will not work meaningfully.
- **Trait/interface bounds** on generic parameters not enforced.
- **Overload resolution** limited to trivial cases.
- **Cross-module forward references** are fragile; circular modules fail.
- **Implicit conversions** beyond widening are inconsistent.

**Fix:** Split `Sema.cpp` into `NameResolver`, `TypeChecker`,
`ExhaustivenessChecker`, `MutabilityChecker`. Tackle mutability and
exhaustiveness first — they're small, high-value, and they unblock real
language use.

---

## HIGH — Will hurt you within weeks

### H1. Parser error recovery
[Source/Parser.cpp](Source/Parser.cpp) (≈L150–L170). `Synchronize()` only resyncs
at keywords/`}`/`;`. A missing semicolon cascades into a dozen secondary errors.
Add token insertion recovery, cap reported errors at ~20, and attach a context
stack to messages.

### H2. No incremental compilation
[Source/Cli.cpp](Source/Cli.cpp). Every `rux build` recompiles the world.
Manifest data exists but isn't used for invalidation. Hash sources + imports,
cache `.rcu`, key on the compiler version.

### H3. Compiler is one monolithic binary
[CMakeLists.txt](CMakeLists.txt). There's no `librux` — a future LSP, formatter,
or doc tool would have to fork the executable. Extract a static library and
make the CLI a thin wrapper.

### H4. No debug information
DWARF / PDB are absent end-to-end. Stack traces are useless, gdb/lldb can't
step Rux code. Plan this alongside the move to `lld` (C5) so DWARF emission
piggybacks on real object files.

### H5. Package manager is a stub
[Source/Cli.cpp](Source/Cli.cpp) — `add`, `install`, `update`, `doc`, `fmt`,
`test` have TODO bodies. No `Rux.lock`, no version range solving, no registry,
no transitive resolution. Pick a minimum viable path: lockfile + path/git deps
+ semver caret ranges + a simple greedy resolver.

---

## MEDIUM — Quality / correctness

### M1. Lexer Unicode hygiene
[Source/Lexer.cpp](Source/Lexer.cpp) (≈L150–L200). UTF-8 is decoded but not
validated; identifier rules don't follow UAX#31; no NFC normalization. Easy
wins, mostly mechanical.

### M2. ABI breadth
[Source/Asm.cpp](Source/Asm.cpp) supports only SysV-AMD64 and Win64. No ARM64
backend — which means no native Apple Silicon, no Linux ARM servers, no mobile.
Plan this only after C1/C2 land, but the LIR design must be checked for
ARM-friendliness now.

### M3. Phi lowering inefficiency
Phi moves at block boundaries are emitted as unconditional copies between
stack slots. A standard "parallel copy" scheme + the new register allocator
removes most of these.

### M4. CLI UX
Diagnostics are line/col but lack snippets, carets, or color in many paths.
Adopt a single `Diagnostic` type (file/range/severity/message/notes) and route
all errors through it; only then add `--format=json` for editor integration.

### M5. Documentation
[docs/](docs/) has examples and a (presumably outdated) gap analysis; there is
no language reference, no spec, no contributor guide describing the IR
contracts. Make the IR contracts (HIR↔LIR) explicit before bringing in new
contributors.

---

## ARCHITECTURAL RISKS — Things that will trap you later

1. **Single-file Sema and Asm.** Splitting later is harder than splitting now.
2. **Custom binary formats.** Every non-standard format becomes a tax on every
   future feature (debug info, LTO, profiling, sanitizers).
3. **No SSA invariants documented for LIR.** Without dominance / phi rules
   written down, every backend pass will reinvent assumptions.
4. **HIR mostly mirrors AST.** Its value is unclear; either give it real
   lowering responsibilities (closures, pattern compilation, generics
   monomorphization) or remove it.
5. **No versioning of `.rcu`.** A cached object from an older compiler will
   silently be reused. Add a magic + version header before anyone uses
   incremental builds.

---

## Recommended Fix-First Roadmap

**Sprint 1 — Foundations (do not skip):**
1. Stand up a real test suite + CI (C6).
2. Split `Sema.cpp`; add mutability + match exhaustiveness (C7).
3. Add slice bounds checking in codegen (C4, partial).
4. Extract `librux` static library (H3).

**Sprint 2 — Backend sanity:**
5. Introduce `Lir::PassManager` + DCE + ConstFold + CopyProp (C2).
6. Implement linear-scan register allocator (C1).
7. Improve parser error recovery and diagnostics (H1, M4).

**Sprint 3 — Ecosystem unblock:**
8. Switch to standard object files + `lld` for linking (C5).
9. Emit DWARF debug info (H4).
10. Ship a minimal `Std` (Io, Mem, Option, Result, Vec) (C3).

**Sprint 4 — Real projects:**
11. Incremental compilation with hashed `.rcu` cache + version header (H2).
12. Package manager v1: lockfile + semver + path/git deps (H5).
13. Decide and start the memory-safety model (C4, full).

Everything beyond Sprint 4 (ARM64 backend, generics monomorphization, async,
macros, LSP) is downstream of getting the above right.
