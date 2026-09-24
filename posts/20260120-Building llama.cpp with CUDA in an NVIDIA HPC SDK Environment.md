---
title: "Building llama.cpp with CUDA in an NVIDIA HPC SDK Environment"
description: "Configure CMake to enable the CUDA backend while selecting GCC/G++ for host code and NVCC for GPU code to avoid compiler-flag incompatibilities."
pubDatetime: 2026-01-20T03:03:14.797Z
---

If you are working in an NVIDIA HPC SDK environment and want to build **llama.cpp** with **CUDA support**, one reliable approach is to use **GCC/G++ for the C/C++ parts** and **NVCC for the CUDA parts**.

This setup is practical because some compiler warning flags used by projects like ggml/llama.cpp are commonly supported by GCC/Clang, but may not be accepted by other C++ compilers. By explicitly selecting `gcc` and `g++`, you reduce the risk of compiler-flag incompatibilities, while still enabling CUDA with `nvcc`.

### Recommended CMake Commands

Run the following commands from the project root directory:

```bash
cmake -S . -B build \
  -DGGML_CUDA=ON \
  -DCMAKE_C_COMPILER=gcc \
  -DCMAKE_CXX_COMPILER=g++ \
  -DCMAKE_CUDA_COMPILER=nvcc

cmake --build build -j
```

### What These Options Do

* `-DGGML_CUDA=ON`
  Enables CUDA support in the ggml backend used by llama.cpp.

* `-DCMAKE_C_COMPILER=gcc` and `-DCMAKE_CXX_COMPILER=g++`
  Ensures the C and C++ source files are compiled with GCC/G++, which typically supports the warning and build flags used by many open-source projects.

* `-DCMAKE_CUDA_COMPILER=nvcc`
  Uses NVIDIA’s CUDA compiler for CUDA code, ensuring proper GPU compilation.

* `cmake --build build -j`
  Builds the project using parallel jobs (the build system decides how many), which usually speeds up compilation.

### Summary

In an NVIDIA HPC SDK environment, explicitly selecting **gcc/g++** for host compilation and **nvcc** for CUDA compilation is a simple and effective way to build llama.cpp with CUDA enabled. This approach is also easy to reproduce across systems and tends to avoid compiler option conflicts.
