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
- Never run `Makefile.deprecated clean`; it deletes generated parser sources required by CMake.
