LLVM with MLGO enabled for Windows

This repository packages a Windows clang release that was built with MLGO support and documents how to pair it with the pretrained MLGO inlining model.

## Downloads

Use these direct links for the exact release artifacts:

- [LLVM MLGO Windows release `22.1.2`](https://github.com/promptengineer1768/llvm-mlgo-build/releases/tag/22.1.2)
- [clang-mlgo-windows.zip](https://github.com/promptengineer1768/llvm-mlgo-build/releases/download/22.1.2/clang-mlgo-windows.zip)
- [MLGO inlining model `inlining-Oz-v1.2`](https://github.com/google/ml-compiler-opt/releases/tag/inlining-Oz-v1.2)
- [saved_model.zip](https://github.com/google/ml-compiler-opt/releases/download/inlining-Oz-v1.2/saved_model.zip)

This repository also publishes an AOT-enabled build that embeds the release-mode inliner model:

- [MLGO AOT build `22.1.2-mlgo-aot`](https://github.com/promptengineer1768/llvm-mlgo-build/releases/tag/22.1.2-mlgo-aot)
- `clang-mlgo-windows-aot.zip` (asset in the release above)

## What You Need

You need two separate downloads:

- the MLGO-enabled compiler release zip
- the pretrained MLGO inlining model zip

After extraction, the compiler lives in one folder and the model lives in another folder. Do not point LLVM at the `.zip` file itself. LLVM needs the extracted `saved_model` directory.

## Suggested Layout

The examples below assume this local structure:

```text
C:\Users\you\Documents\Projects\llvm\
  downloads\
    clang-mlgo-windows.zip
    clang-mlgo-windows\
      clang++.exe
      clang-cl.exe
      clang.exe
    saved_model.zip
    inlining-Oz-v1.2\
      saved_model\
        saved_model.pb
        model.tflite
        policy_specs.pbtxt
        output_spec.json
        variables\
    clang-mlgo-windows-aot.zip
    clang-mlgo-windows-aot\
      bin\
      include\
      lib\
      share\
```

## Setup Steps

1. Download [clang-mlgo-windows.zip](https://github.com/promptengineer1768/llvm-mlgo-build/releases/download/22.1.2/clang-mlgo-windows.zip).
2. Extract it into `C:\Users\you\Documents\Projects\llvm\downloads\clang-mlgo-windows`.
3. Download [saved_model.zip](https://github.com/google/ml-compiler-opt/releases/download/inlining-Oz-v1.2/saved_model.zip).
4. Extract it into `C:\Users\you\Documents\Projects\llvm\downloads\inlining-Oz-v1.2`.
5. Confirm the model directory exists at `C:\Users\you\Documents\Projects\llvm\downloads\inlining-Oz-v1.2\saved_model`.
6. Add `C:\Users\you\Documents\Projects\llvm\downloads\clang-mlgo-windows` to `PATH`, or invoke `clang++.exe` and `clang-cl.exe` with full paths.
7. If `clang-cl.exe` does not start on your machine, make sure the MinGW runtime DLLs it depends on are available on `PATH`. On this machine, `C:\Program Files\Git\mingw64\bin` satisfies those dependencies.

If you want a compiler that already embeds the release-mode MLGO model, download and extract the AOT release:

1. Download `clang-mlgo-windows-aot.zip` from the `22.1.2-mlgo-aot` release.
2. Extract it into `C:\Users\you\Documents\Projects\llvm\downloads\clang-mlgo-windows-aot`.

## Using The Model

The pretrained model is used when you build LLVM from source with MLGO enabled and point LLVM at the extracted model directory.

```powershell
$env:LLVM_INLINER_MODEL_PATH = "C:\Users\you\Documents\Projects\llvm\downloads\inlining-Oz-v1.2\saved_model"

cmake -GNinja `
  -S <path-to-llvm-source> `
  -B <build-dir> `
  -DLLVM_ENABLE_PROJECTS="clang;lld" `
  -DLLVM_ENABLE_MLGO=ON `
  -DLLVM_INLINER_MODEL_PATH="$env:LLVM_INLINER_MODEL_PATH"
```

If you want to verify the extracted files before building, the model directory should contain `saved_model.pb`, `model.tflite`, `policy_specs.pbtxt`, `output_spec.json`, and `variables\`.

## Benchmark

This repository includes a small benchmark that creates inlining pressure in several hot helper functions:

- [bench/mlgo_inline_bench.cpp](bench/mlgo_inline_bench.cpp)

The benchmark is intentionally simple so you can compare a baseline compiler build against an MLGO-enabled build using the same source and the same input size.

Build and run the benchmark with the MLGO compiler:

```powershell
$clang = "C:\Users\you\Documents\Projects\llvm\downloads\clang-mlgo-windows\clang++.exe"

& $clang -O2 -std=c++20 bench\mlgo_inline_bench.cpp -o bench-mlgo.exe
Measure-Command { .\bench-mlgo.exe }
```

For a fair comparison, compile the same source with your baseline compiler and compare the reported `elapsed_ms` value under the same number of rounds. If you are building LLVM itself with MLGO, use the extracted model directory from this README before rebuilding the compiler.

## Release-Mode Tutorial (AOT Build)

This tutorial uses the AOT build (`22.1.2-mlgo-aot`) and shows how to compile a small inlining benchmark in both baseline mode and MLGO release mode. The steps are validated against the AOT release artifact.

### 1) Extract the AOT release

```powershell
Expand-Archive -Path "C:\Users\you\Documents\Projects\llvm\downloads\clang-mlgo-windows-aot.zip" `
  -DestinationPath "C:\Users\you\Documents\Projects\llvm\downloads\clang-mlgo-windows-aot" -Force
```

### 2) Build baseline and MLGO binaries

The AOT release ships `clang-cl.exe`, so the easiest reproducible path is to use `vcvars64.bat` and `clang-cl`.

```cmd
cmd /c "call \"C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat\" >nul ^
 && set PATH=C:\Program Files\Git\mingw64\bin;%PATH% ^
 && \"C:\Users\you\Documents\Projects\llvm\downloads\clang-mlgo-windows-aot\bin\clang-cl.exe\" /nologo /std:c++20 /O2 ^
    \"C:\Users\you\Documents\Projects\llvm\llvm-mlgo-build\bench\mlgo_inline_bench.cpp\" ^
    /Fe:\"C:\Users\you\Documents\Projects\llvm\downloads\bench-baseline.exe\""
```

```cmd
cmd /c "call \"C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat\" >nul ^
 && set PATH=C:\Program Files\Git\mingw64\bin;%PATH% ^
 && \"C:\Users\you\Documents\Projects\llvm\downloads\clang-mlgo-windows-aot\bin\clang-cl.exe\" /nologo /std:c++20 /O2 ^
    \"C:\Users\you\Documents\Projects\llvm\llvm-mlgo-build\bench\mlgo_inline_bench.cpp\" ^
    /Fe:\"C:\Users\you\Documents\Projects\llvm\downloads\bench-mlgo.exe\" ^
    /clang:-mllvm /clang:-enable-ml-inliner=release"
```

### 3) Run and compare

The benchmark prints `elapsed_ms`. For a steadier signal, increase the rounds count:

```powershell
& "C:\Users\you\Documents\Projects\llvm\downloads\bench-baseline.exe" 800
& "C:\Users\you\Documents\Projects\llvm\downloads\bench-mlgo.exe" 800
```

If you want a heavier inlining stress test, you can also compile and run:

```cmd
cmd /c "call \"C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat\" >nul ^
 && set PATH=C:\Program Files\Git\mingw64\bin;%PATH% ^
 && \"C:\Users\you\Documents\Projects\llvm\downloads\clang-mlgo-windows-aot\bin\clang-cl.exe\" /nologo /std:c++20 /O2 ^
    \"C:\Users\you\Documents\Projects\llvm\src\mlgo_bench.cpp\" ^
    /Fe:\"C:\Users\you\Documents\Projects\llvm\downloads\mlgo-bench-baseline.exe\""
```

```cmd
cmd /c "call \"C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat\" >nul ^
 && set PATH=C:\Program Files\Git\mingw64\bin;%PATH% ^
 && \"C:\Users\you\Documents\Projects\llvm\downloads\clang-mlgo-windows-aot\bin\clang-cl.exe\" /nologo /std:c++20 /O2 ^
    \"C:\Users\you\Documents\Projects\llvm\src\mlgo_bench.cpp\" ^
    /Fe:\"C:\Users\you\Documents\Projects\llvm\downloads\mlgo-bench-mlgo.exe\" ^
    /clang:-mllvm /clang:-enable-ml-inliner=release"
```
