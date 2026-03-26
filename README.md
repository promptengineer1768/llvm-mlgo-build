LLVM with MLGO enabled for Windows

This repository packages a Windows clang release that was built with MLGO support and documents how to pair it with the pretrained MLGO inlining model.

## Downloads

Use these direct links for the exact release artifacts:

- [LLVM MLGO Windows release `22.1.2`](https://github.com/promptengineer1768/llvm-mlgo-build/releases/tag/22.1.2)
- [clang-mlgo-windows.zip](https://github.com/promptengineer1768/llvm-mlgo-build/releases/download/22.1.2/clang-mlgo-windows.zip)
- [MLGO inlining model `inlining-Oz-v1.2`](https://github.com/google/ml-compiler-opt/releases/tag/inlining-Oz-v1.2)
- [saved_model.zip](https://github.com/google/ml-compiler-opt/releases/download/inlining-Oz-v1.2/saved_model.zip)

## What You Need

You need two separate downloads:

- the MLGO-enabled compiler release zip
- the pretrained MLGO inlining model zip

After extraction, the compiler lives in one folder and the model lives in another folder. Do not point LLVM at the `.zip` file itself. LLVM needs the extracted `saved_model` directory.

## Suggested Layout

The examples below assume this local structure:

```text
C:\Users\me\Documents\OpenCode projects\llvm\
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
```

## Setup Steps

1. Download [clang-mlgo-windows.zip](https://github.com/promptengineer1768/llvm-mlgo-build/releases/download/22.1.2/clang-mlgo-windows.zip).
2. Extract it into `C:\Users\me\Documents\OpenCode projects\llvm\downloads\clang-mlgo-windows`.
3. Download [saved_model.zip](https://github.com/google/ml-compiler-opt/releases/download/inlining-Oz-v1.2/saved_model.zip).
4. Extract it into `C:\Users\me\Documents\OpenCode projects\llvm\downloads\inlining-Oz-v1.2`.
5. Confirm the model directory exists at `C:\Users\me\Documents\OpenCode projects\llvm\downloads\inlining-Oz-v1.2\saved_model`.
6. Add `C:\Users\me\Documents\OpenCode projects\llvm\downloads\clang-mlgo-windows` to `PATH`, or invoke `clang++.exe` and `clang-cl.exe` with full paths.
7. If `clang-cl.exe` does not start on your machine, make sure the MinGW runtime DLLs it depends on are available on `PATH`. On this machine, `C:\Program Files\Git\mingw64\bin` satisfies those dependencies.

## Using The Model

The pretrained model is used when you build LLVM from source with MLGO enabled and point LLVM at the extracted model directory.

```powershell
$env:LLVM_INLINER_MODEL_PATH = "C:\Users\me\Documents\OpenCode projects\llvm\downloads\inlining-Oz-v1.2\saved_model"

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
$clang = "C:\Users\me\Documents\OpenCode projects\llvm\downloads\clang-mlgo-windows\clang++.exe"

& $clang -O2 -std=c++20 bench\mlgo_inline_bench.cpp -o bench-mlgo.exe
Measure-Command { .\bench-mlgo.exe }
```

For a fair comparison, compile the same source with your baseline compiler and compare the reported `elapsed_ms` value under the same number of rounds. If you are building LLVM itself with MLGO, use the extracted model directory from this README before rebuilding the compiler.
