---
title: "How to Build NCCL Tests in an NVIDIA HPC SDK Environment"
description: "Point the nccl-tests build at the SDK's CUDA, NCCL, and associated HPC-X Open MPI directories, then enable MPI support through explicit make variables."
pubDatetime: 2026-04-14T08:23:44.992Z
---

To build `nccl-tests` in an NVIDIA HPC SDK environment, you need to set the correct paths for CUDA, NCCL, and MPI, and then run `make`. This article briefly explains the required environment variables and the build command.

## Overview

First, define the locations of CUDA and NCCL based on your NVIDIA HPC SDK installation. Then set `MPI_HOME` to the Open MPI bundled with HPC-X alongside NCCL.

```bash
export CUDA_HOME="$NVHPC_ROOT/cuda"
export NCCL_HOME="$NVHPC_ROOT/comm_libs/nccl"
export MPI_HOME="$(readlink -f "$NCCL_HOME")/../hpcx/latest/ompi"
make -kj MPI=1 CUDA_HOME="$CUDA_HOME" NCCL_HOME="$NCCL_HOME" MPI_HOME="$MPI_HOME"
```

## Meaning of Each Environment Variable

### `CUDA_HOME`

`CUDA_HOME` points to the CUDA directory inside the NVIDIA HPC SDK installation.

```bash
export CUDA_HOME="$NVHPC_ROOT/cuda"
```

This allows the build process to find the correct CUDA headers and libraries.

### `NCCL_HOME`

`NCCL_HOME` specifies the location of the NCCL library included in the HPC SDK.

```bash
export NCCL_HOME="$NVHPC_ROOT/comm_libs/nccl"
```

`nccl-tests` uses this path to locate the NCCL installation.

### `MPI_HOME`

`MPI_HOME` points to the Open MPI installation provided through HPC-X near the NCCL package.

```bash
export MPI_HOME="$(readlink -f "$NCCL_HOME")/../hpcx/latest/ompi"
```

Using `readlink -f` resolves the symbolic link and ensures that the path is based on the actual NCCL installation directory.

## Build Command

After setting the environment variables, run the following command to build `nccl-tests`.

```bash
make -kj MPI=1 CUDA_HOME="$CUDA_HOME" NCCL_HOME="$NCCL_HOME" MPI_HOME="$MPI_HOME"
```

### Key Points

* `MPI=1`
  Enables MPI support during the build.
* `CUDA_HOME=...`
  Explicitly sets the CUDA location.
* `NCCL_HOME=...`
  Explicitly sets the NCCL location.
* `MPI_HOME=...`
  Specifies which MPI implementation to use.
* `-kj`
  Enables parallel build execution.

## Summary

To build `nccl-tests` in an NVIDIA HPC SDK environment, the main requirement is to correctly define `CUDA_HOME`, `NCCL_HOME`, and `MPI_HOME`, and pass them to `make`. A notable point is that `MPI_HOME` should be set to the HPC-X Open MPI installation associated with NCCL. With this setup, `nccl-tests` can be built cleanly within the HPC SDK environment.
