# shecc Repository Guide for Agents

## Project Overview
- `shecc` is a self-hosting educational C compiler that targets 32-bit Armv7-A and RV32IM (RISC-V) Linux userland binaries.  The compiler, runtime libc, build tools, and extensive regression tests live in this repository.
- The default build flow produces three compilers:
  1. **Stage 0** – a host binary compiled with the system C compiler.
  2. **Stage 1** – an Arm/RISC-V binary emitted by stage 0.
  3. **Stage 2** – a binary produced by running stage 1 under QEMU/on target hardware; it must match stage 1 byte-for-byte for a successful bootstrap (`make bootstrap`).

## Repository Layout
- `src/` – compiler sources.  `main.c` textually includes other `.c` files; keep the include order intact.  Do not add a tracked `src/codegen.c`; it is generated as a symlink by `make config`.
- `lib/c.c` – the embedded libc that gets inlined into outputs (`out/libc.inc`).
- `mk/*.mk` – build recipes per backend (`arm.mk`, `riscv.mk`) and shared rules (`common.mk`).  `make config ARCH=<arm|riscv>` writes `./config` and links `src/codegen.c`.
- `tools/` – helper utilities built during `make` (`norm-lf`, `inliner`).
- `tests/` – 200+ compiler regression programs (`driver.sh` orchestrates staged execution).  Snapshot baselines reside in `tests/snapshots/`.
- `.ci/` – formatting (`check-format.sh`) and newline (`check-newline.sh`) linters used in CI.

Generated artifacts (`out/`, `config`, `src/codegen.c`, `.session.mk`, test logs, etc.) are git-ignored; never commit them.

## Tooling & Prerequisites
- Host build relies on a modern C compiler (gcc/clang), GNU make, and standard Unix tooling.
- Tests and snapshots require `qemu-user`, `graphviz`, and `jq` (see `README.md`).  Ensure `qemu-arm` or `qemu-riscv32` is installed so staged compilers can execute.
- Coding style is enforced with `clang-format` **18** (`.clang-format` is provided).  Install the matching version before formatting.

## Build & Verification Workflow
1. Select a backend before building: `make distclean config ARCH=arm` (or `ARCH=riscv`).  Re-run whenever you switch targets.
2. `make` (without arguments) builds stage0/1/2 and verifies the bootstrap.
3. For sanitizers: `make sanitizer` then `make check-sanitizer` (AddressSanitizer + UBSan).
4. IR tooling:
   - `make check-snapshot` checks the current backend’s IR snapshots.
   - `make check-snapshots` iterates all backends.
   - `make update-snapshot` / `make update-snapshots` refresh baselines when intentionally changing the frontend/IR.  Review diffs carefully before committing.
5. Cleanups: `make clean` removes build results for the active backend; `make distclean` purges generated helpers (`out/inliner`, `out/norm-lf`, `config`, the codegen link, etc.).

## Testing Expectations
Run the following after code changes (pick commands according to what you touched):
- Always run `make check` for compiler, runtime (`lib/`), or tool (`tools/`) modifications.
- If you changed lexer/parser/IR/backends (`src/`), also run `make check-snapshot` for the currently configured architecture; update snapshots only when differences are intended.
- If you touched low-level code that could affect memory safety (allocator, libc, code generation, register allocation, etc.), run `make check-sanitizer`.
- For documentation-only edits, tests may be skipped; call this out explicitly in the final summary.
- Formatting/newline checks: when you modify C/C++ headers or sources, format them with `clang-format-18 -i` and verify with `.ci/check-format.sh`.  Ensure all tracked files end with a newline using `.ci/check-newline.sh`.

Document the exact commands you executed (or why they were skipped) in the final report.

## Coding Style Highlights
- Follow the K&R-derived style in `CONTRIBUTING.md`: 4-space indentation, no tabs, 80-character guideline, and consistent spacing around operators and keywords.
- Use snake_case identifiers; avoid camelCase, Hungarian notation, and redundant parentheses.
- Place braces per Linux/K&R conventions.  Keep `if/else` blocks balanced; prefer early returns to deep nesting.
- Always parenthesize macro parameters; wrap multi-statement macros in `do { ... } while (0)`.
- Prefer `const`/`static` qualifiers where applicable, and use `size_t`/`ssize_t` for sizes.
- Treat portability seriously: do not assume specific endianness or integer widths; rely on fixed-width types only when intentional.

## Additional Tips
- `config` is a generated header containing backend-specific defines; include guards are handled automatically.  Never edit it manually.
- Snapshot failures typically mean the frontend/optimizer changed IR output—inspect `tests/snapshots/*` diffs.
- `tests/driver.sh` honors `VERBOSE=1`, `SHOW_PROGRESS=0`, etc., which can help diagnose failures locally.
- When running inside CI or containers, ensure `TARGET_EXEC` (set by `make config` when QEMU is required) points to the proper emulator path.
- Keep documentation in American English as requested in `CONTRIBUTING.md`.

This file applies to the entire repository.
