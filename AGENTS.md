# AGENTS.md

## Cursor Cloud specific instructions

This repo is **OLLVM-16**: an out-of-tree LLVM 16 obfuscation pass plugin (C++). There is no
web server, database, package manifest, lockfile, lint config, or automated test suite. The
"application" is the compiler plugin; "running" it means building it into a loadable pass
plugin and using it to obfuscate a compiled program. See `README.md` for the upstream in-tree
build method and the `-sub` / `-split` / `-fla` / `-bcf` usage flags.

### Toolchain (baked into the VM snapshot)
- LLVM 16.0.4 prebuilt toolchain at `/opt/llvm16` (provides `clang`/`clang++` 16 and the
  `lib/cmake/llvm` modules, incl. `AddLLVM.cmake`). The system's default `clang` is 18 and is
  **API-incompatible** — always use `/opt/llvm16/bin` for anything touching this plugin.
- System build deps: `ninja-build`, `g++`, `libstdc++-14-dev` (clang-16 selects the gcc-14
  toolchain, so its libstdc++ dev package must be present or linking fails with `-lstdc++`).

### Build (out-of-tree, avoids a multi-hour full LLVM source build)
A CMake wrapper lives OUTSIDE the repo at `~/ollvm-build` (kept out of the repo so nothing here
is modified). It provides a stand-in `intrinsics_gen` target and does
`add_subdirectory(/workspace/Obfuscation)`. Build with:

```
ninja -C ~/ollvm-build/build
```

Output plugin: `~/ollvm-build/build/Obfuscation/Obfuscation.so`. Reconfigure (e.g. after a
clean) with:

```
cmake -G Ninja -B ~/ollvm-build/build -S ~/ollvm-build \
  -DCMAKE_BUILD_TYPE=Release -DLLVM_DIR=/opt/llvm16/lib/cmake/llvm \
  -DCMAKE_C_COMPILER=/opt/llvm16/bin/clang -DCMAKE_CXX_COMPILER=/opt/llvm16/bin/clang++ \
  -DOLLVM_SOURCE_DIR=/workspace/Obfuscation
```

### Running / testing the obfuscator (important gotcha)
The plugin registers its `-mllvm` flags (`-sub`,`-split`,`-fla`,`-bcf`) as `cl::opt` that only
exist once the `.so` is loaded. With `-fpass-plugin=...` alone, clang parses `-mllvm` flags
*before* it dlopens the plugin, so you get `Unknown command line argument '-sub'`. Workaround:
`LD_PRELOAD` the plugin so its `cl::opt` register at process start, but do it **only for the
compile step** — `LD_PRELOAD` leaks into the linker (`ld`) and breaks linking with an LLVM
symbol-lookup error. So compile to an object first, then link separately:

```
PLUGIN=~/ollvm-build/build/Obfuscation/Obfuscation.so
LD_PRELOAD=$PLUGIN /opt/llvm16/bin/clang -fpass-plugin=$PLUGIN \
  -mllvm -sub -mllvm -split -mllvm -fla -mllvm -bcf -c prog.c -o prog.o
/opt/llvm16/bin/clang prog.o -o prog     # no LD_PRELOAD here
```

(The upstream alternative is `-DLLVM_OBFUSCATION_LINK_INTO_TOOLS=ON`, which statically links the
passes into clang, but that requires a full in-tree LLVM build — impractical on this 4-CPU VM.)

To verify obfuscation actually happened, emit IR and compare (`-S -emit-llvm` needs no linker,
so `LD_PRELOAD` on that single command is fine): flattening/bogus-control-flow multiply the
basic-block and `switch` counts vs. a `-O0 -disable-O0-optnone` baseline, while program output
stays identical. A ready sample is at `~/ollvm-build/test/sample.c`.
