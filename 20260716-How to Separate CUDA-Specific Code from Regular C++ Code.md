---
title: "How to Separate CUDA-Specific Code from Regular C++ Code"
description: "In a CUDA application, you can place GPU kernels and device memory-related code in .cu files while..."
pubDatetime: 2026-07-16T11:06:32.591Z
updatedDate: 2026-07-16T11:34:54.682Z
---

In a CUDA application, you can place GPU kernels and device memory-related code in `.cu` files while keeping ordinary C++ logic in `.cpp` files.

This structure helps prevent CUDA-specific code from being mixed with application logic, making each part of the codebase easier to understand and maintain.

## File Organization

A typical separation of responsibilities looks like this:

* `cuda.cu`

  * CUDA kernels
  * `__constant__` memory
  * CUDA-specific functions
* `host.cpp`

  * `main` function
  * Input/output
  * Memory allocation
  * Kernel launches
  * Result verification

## CUDA-Side Implementation

On the CUDA side, define the constant memory and the kernel.

```cpp
// cuda.cu

#include <cstdint>

// Constant memory with one element
__constant__ std::uint32_t constant_value[1];

// Kernel that copies the constant memory value to global memory
__global__ void copyConstantToGlobal(std::uint32_t* destination)
{
    if (blockIdx.x == 0 && threadIdx.x == 0) {
        destination[0] = constant_value[0];
    }
}
```

`constant_value` stores the value that will be accessed from the GPU.

The `copyConstantToGlobal` kernel copies the value stored in constant memory into global memory.

## Referencing CUDA Symbols from C++

A standard C++ compiler cannot process CUDA-specific keywords such as `__constant__` and `__global__`.

Therefore, declare these symbols in the C++ source without CUDA qualifiers.

```cpp
// host.cpp

#include <cstdint>

extern std::uint32_t constant_value[1];

void copyConstantToGlobal(std::uint32_t* destination);
```

These are declarations, not definitions. They simply reference symbols that are defined in another translation unit.

## Writing a Value to Constant Memory

To write a value from the host to constant memory, first obtain the device address with `cudaGetSymbolAddress`.

```cpp
void* constant_address = nullptr;

CUDA_CHECK(cudaGetSymbolAddress(
    &constant_address,
    constant_value
));
```

Then copy the value to that address using `cudaMemcpy`.

```cpp
CUDA_CHECK(cudaMemcpy(
    constant_address,
    &input_value,
    sizeof(input_value),
    cudaMemcpyHostToDevice
));
```

This stores the host-generated value in `constant_value[0]`.

## Allocating Global Memory for Output

Allocate global memory on the GPU to hold the kernel output.

```cpp
std::uint32_t* device_output = nullptr;

CUDA_CHECK(cudaMalloc(
    reinterpret_cast<void**>(&device_output),
    sizeof(*device_output)
));
```

This allocates enough space for a single `std::uint32_t`.

## Launching the Kernel with `cudaLaunchKernel`

Normally, a kernel is launched inside a `.cu` file using the following syntax:

```cpp
copyConstantToGlobal<<<1, 1>>>(device_output);
```

However, the `<<<...>>>` launch syntax is not supported by a standard C++ compiler.

When launching a kernel from a `.cpp` file, use `cudaLaunchKernel` instead.

### Preparing the Kernel Arguments

`cudaLaunchKernel` expects an array containing the addresses of host variables that hold the kernel arguments.

In this example, the kernel has a single argument:

```cpp
std::uint32_t* destination
```

Since `device_output` is the pointer value to be passed to the kernel, register the address of that variable, `&device_output`.

```cpp
void* kernel_args[] = {
    &device_output
};
```

### Launching the Kernel

```cpp
const dim3 grid_dim(1, 1, 1);
const dim3 block_dim(1, 1, 1);

CUDA_CHECK(cudaLaunchKernel(
    reinterpret_cast<const void*>(copyConstantToGlobal),
    grid_dim,
    block_dim,
    kernel_args,
    0,
    nullptr
));
```

The arguments are as follows:

* `copyConstantToGlobal`

  * The kernel to launch.
* `grid_dim`

  * Grid dimensions.
* `block_dim`

  * Block dimensions.
* `kernel_args`

  * Kernel arguments.
* `0`

  * Size of dynamically allocated shared memory.
* `nullptr`

  * Use the default stream.

Since this example copies only a single value, the kernel is launched with one block containing one thread.

## Waiting for Kernel Completion

After launching the kernel, check for launch errors and wait for execution to complete.

```cpp
CUDA_CHECK(cudaGetLastError());
CUDA_CHECK(cudaDeviceSynchronize());
```

Kernel launches through `cudaLaunchKernel` are asynchronous. Before reading back the result, call `cudaDeviceSynchronize` to ensure the GPU has finished execution.

## Copying the Result Back to the Host

Copy the output value from GPU memory back to the host.

```cpp
std::uint32_t output_value = 0;

CUDA_CHECK(cudaMemcpy(
    &output_value,
    device_output,
    sizeof(output_value),
    cudaMemcpyDeviceToHost
));
```

You can verify correct operation by comparing the written value with the value read back.

```cpp
const bool succeeded = input_value == output_value;

std::cout << "Result        : "
          << (succeeded ? "PASS" : "FAIL")
          << std::endl;
```

Finally, release the allocated GPU memory.

```cpp
CUDA_CHECK(cudaFree(device_output));
```

## Compiling

Compile CUDA source files with `nvcc` and regular C++ source files with `g++`.

```bash
nvcc -rdc=true -c cuda.cu
g++ -c host.cpp
nvcc cuda.o host.o -o constant_symbol_test
```

It is important to perform the final linking step with `nvcc`. Using `nvcc` ensures that the CUDA runtime and device code are linked correctly.

The `-rdc=true` option enables separate compilation of CUDA code across multiple translation units.

## Purpose of This Structure

Separating CUDA code from standard C++ code provides several benefits:

* Clearly separates GPU processing from host-side logic.
* Allows `.cpp` files to be compiled with a standard C++ compiler.
* Prevents CUDA-specific syntax from spreading throughout the application.
* Makes the CUDA implementation easier to manage as an independent module.
* Simplifies integrating CUDA into an existing C++ project.

The key idea is to place CUDA-specific definitions in `.cu` files while declaring only the required symbols in standard C++ form on the host side.

Instead of using the `<<<...>>>` launch syntax, launch kernels from `.cpp` files with `cudaLaunchKernel`, and perform the final linking step with `nvcc`.
