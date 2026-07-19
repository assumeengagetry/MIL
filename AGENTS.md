# Repository Instructions

## Build

- CMake is authoritative; `Makefile.deprecated` does not build the application.
- Building requires CMake 3.16.4+, a C++11 compiler, and Readline development headers. Bison and Flex are needed only when regenerating the parser.
- `compiler/asmjit` is an empty gitlink pinned to `8e1eb4f5dc052496295f60d5c9f28c5b6d347ae7`, but this repository has no `.gitmodules`; `git submodule update --init` fails. Restore the compatible old API with:
  ```bash
  git clone https://github.com/asmjit/asmjit.git compiler/asmjit
  git -C compiler/asmjit checkout --detach 8e1eb4f5dc052496295f60d5c9f28c5b6d347ae7
  ```
- The repository has no `.gitignore`; keep build output outside the worktree:
  ```bash
  cmake -S . -B /tmp/mil-build
  cmake --build /tmp/mil-build --target matbar --parallel
  ```
- `CMakeLists.txt` has an explicit source list. Add new production `.cc` files there; source discovery is not automatic.

## Architecture

- `common/Main.cc` repeatedly calls `yyparse()`. Grammar actions build stack IR through the global services declared in `include/RuntimePool.h` and defined in `IR/RuntimePool.cc`.
- `parser/` owns syntax, `IR/` owns generation and the variable/function registry, `IRVM/IRExecution.cc` interprets expressions and assignments, and `compiler/` optimizes and JIT-compiles function definitions.
- Operator or call semantics span two execution paths: the interpreter in `IRVM/IRExecution.cc` and the AsmJit translator in `compiler/IRTranslator.cc`. Keep both paths aligned.
- `VFManager` owns `IRVariable` and `IRFunction` raw pointers. JIT code embeds global-variable addresses and current callee addresses, so deletion or redefinition can leave compiled functions dangling.
- Treat `CMakeLists.txt`, `parser/parser.y`, `parser/scanner.l`, and `common/Controller.cc`'s `cmdsMap` as authoritative. The language docs describe optimizer commands, built-in math calls, IR dumping, and `@moe` behavior that production code does not fully wire up.
- `IRVM/Moe.cc` is not in the CMake target and is incomplete; adding it directly introduces unresolved functions.

## Run And Verify

- `matbar` ignores CLI arguments and reads MIL from stdin. Every statement must end with `;`.
- There is no CTest target, test framework, output oracle, lint, formatter, typecheck, or CI. Building `matbar` is the compile check.
- Use a focused interpreter smoke test, then `test/test.in` for variables, function JIT compilation, enumeration, and disassembly:
  ```bash
  printf '1+1;\n@quit;\n' | /tmp/mil-build/matbar
  /tmp/mil-build/matbar < test/test.in
  ```
- Files and binaries under `test/` and `benchmark/` are unregistered historical experiments; do not treat them as maintained test targets.

## Parser Generation

- CMake compiles checked-in `parser/y.tab.cc` and `parser/lex.yy.cc`; it never invokes Bison or Flex.
- After changing `parser/parser.y` or `parser/scanner.l`, run `make -C parser all`, then inspect every generated change under `parser/`. The rule also updates `y.tab.h` and `y.output`.
- Checked-in Bison output was generated with 3.5.1, so newer Bison versions can cause broad skeleton churn.
- Scanner semantic values come from `strdup()` and are consumed with `free()` by `IRGenerator` or `Controller`; their `const char *` APIs still transfer ownership.
- `include/Parser.h` defines `YYSTYPE` as `char *`, while standalone generated `y.tab.h` falls back to `int`. Preserve the current include order and inspect this boundary after regeneration.
- Never run `Makefile.deprecated clean`; it deletes generated parser sources required by CMake.
