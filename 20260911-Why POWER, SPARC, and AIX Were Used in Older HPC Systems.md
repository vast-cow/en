---
title: "Why POWER, SPARC, and AIX Were Used in Older HPC Systems"
description: "Broadly speaking, back then, systems using CPUs like POWER and SPARC, along with commercial UNIX,..."
pubDatetime: 2026-09-11T04:49:32.010Z
---

Broadly speaking, **back then, systems using CPUs like POWER and SPARC, along with commercial UNIX, were the ones that provided the necessary performance and operating environment for HPC in a comprehensive package.** Later, systems using mass-produced components and Linux also became capable of delivering the required performance, and offered advantages in terms of price and procurement flexibility. Rather than simply saying that "older systems used specialized components," it's easier to understand if you think of it as a shift in the rational choices available. ([IBM][1])

Here, we will primarily focus on the transition from the 1990s to the 2000s and explain the situation with CPUs and operating systems separately.

## 1. Why CPUs like POWER and SPARC?

### Actively Investing Performance in the Areas Necessary for Scientific and Technological Computing

In HPC, it's not just clock speed that matters. Factors such as how many floating-point operations can be processed, and how quickly data can be supplied from memory to the arithmetic unit, are also important.

High-performance RISC machines at the time were designed to enhance these aspects. For example, IBM's POWER2 featured two floating-point units and increased cache capacity and memory-to-cache transfer bandwidth. The subsequent POWER3 utilized 64-bit address space and large shared memory. It was designed to handle large-scale numerical calculations, including the memory system, not just the CPU itself. ([Wayback Machine][2])

The RISC architecture also played a role here. By organizing the instruction set and efficiently implementing a pipeline that streamlines instruction processing, the goal was to achieve high performance. IBM's RS/6000 was also deployed as a high-performance workstation/server for scientific and engineering applications. ([IBM][1])

However, **it's not accurate to say that "RISC is always faster than x86."** The classification of instruction sets and the actual arithmetic units, cache, and memory bandwidth of a product are different things. Even with the same instruction set, the type of computation it excels at changes depending on how resources like circuits and power are allocated.

The same distinction is important for SPARC. For example, the SPARC64 VIIIfx used in "Kei" wasn't simply a large number of standard SPARC CPUs; it was a CPU developed with a focus on performance, power efficiency, and reliability through error correction and instruction re-execution for scientific and technological computing. The reason it was chosen wasn't simply because it was "SPARC," but because **its specific implementation was well-suited to the system's goals.** ([Fujitsu Archives Information][3])

It should be noted that **IBM's POWER and PowerPC are related series, but they are not strictly the same name.** PowerPC was created based on POWER through collaboration between IBM, Apple, and Motorola. Both appeared in the history of HPC. ([IBM][1])

## 2. Why OSs like AIX instead of Linux?

### You Were Buying the Entire Computing System, Not Just Choosing an OS Freely

This is particularly important.

Today, people tend to think of "first buying a server and then installing Linux," but in older vendor-made HPC systems, the hardware and the software environment that fully utilized it were closely linked.

For example, IBM's SP system included not only AIX but also high-speed inter-node communication, C/Fortran compilers, the ESSL numerical calculation library, a parallel execution environment including MPI, job management with LoadLeveler, and a parallel file system with GPFS. IBM's parallel execution environment at the time depended on AIX and the POWER platform. ([Wayback Machine][2])

Therefore, from the user's perspective, the choice was:

> "Which of AIX or Linux should I use on the same machine?"

rather than

> **"Choosing the combination of hardware, development environment, and operating environment that can perform the required calculations."**

It's more appropriate to see it as a shift in the nature of the choice.

A different OS simply running on the CPU doesn't create an equivalent HPC system. It needs to be usable, including support for communication devices, compiler optimization, parallel execution, and fault diagnosis.

### AIX Had Established Track Record as a Product

AIX has been around since 1986 and is a product that has been used for critical business operations in companies. In HPC, it wasn't just about speed; there was value in being able to operate it continuously and have problems investigated and fixed. Utilizing an existing product base was rational. ([IBM Community][4])

However, this is **not to say that "using AIX makes the OS execute floating-point operations particularly faster."** The CPU performs the core numerical calculations, and the compiler and numerical calculation libraries generate and select the instructions. While the OS has an impact through memory management and communication, the reason for its adoption is not the speed of the OS itself, but the overall performance and completeness.

Also, saying that "Linux at the time couldn't be used for HPC" is an exaggeration. In 1994, the Beowulf project began at NASA, using open-source software like Linux and off-the-shelf components. **Commercial UNIX machines and Linux clusters coexisted and transitioned depending on the application and scale.** ([Beowulf][5])

## 3. Why the Shift to x86 + Linux?

### x86's Numerical Computing Capabilities Were Enhanced

x86 didn't remain the same in terms of performance. For example, SSE2 introduced SIMD instructions for processing double-precision floating-point numbers in batches, and subsequent AVX instructions expanded the arithmetic functions. Functions necessary for scientific and technological computing were incorporated into widely available CPUs. ([Intel][6])

What's important here is that x86 didn't need to be the highest-performing for all applications. If the calculation can be sufficiently parallelized, the criterion is not just "the highest performance of a CPU when purchased," but **how much computation can be done with the entire system for the same budget.**

### The Advantages of Using Mass-Produced Components and a Common Parallel Software Stack Became Significant

If you can use CPUs, memory, and network components supplied in large quantities for the PC market, you can benefit from price competition and economies of scale compared to building an HPC system with dedicated components. Furthermore, the price-performance ratio of networks and the common parallel programming environment such as MPI enhanced the practicality of clusters using mass-produced components. These are also factors in the development of Beowulf. ([Beowulf][5])

However, it wasn't as if "Linux clusters appeared, and that's when people started connecting multiple computers." The IBM SP mentioned earlier was also a parallel system. The key change was that **parallelization was not invented, but rather, it became possible to build parallel machines with more widely available components and common software.** ([Wayback Machine][2])

### Linux Could Inherit UNIX Usage and Become a Common Platform

Linux reimplements UNIX functions and aims for compatibility with POSIX, making it an OS that is easy to adapt to UNIX-based development and operations. It was originally developed for x86 but supports many CPU architectures. ([kernel.org][7])

Being open source was also important. In fact, in early Beowulf, the Linux driver was modified to improve network performance. It wasn't just about saving on license fees; the advantage was that **it could be investigated and modified as needed, and a common environment could be developed by many organizations.** ([Beowulf][5])

## 4. The "Shift to x86" and the "Shift to Linux" Are Different Phenomena

These two progressed together, but they are not the same thing. Examples make this clear:

| System                       | CPU                   | OS                       |
| ---------------------------- | --------------------- | ------------------------ |
| Kei [2011 system]            | SPARC64 VIIIfx        | Linux                    |
| Summit [2018 system]         | IBM POWER9 + NVIDIA GPU | Red Hat Enterprise Linux |

Kei used SPARC but ran Linux, and Summit also used a POWER-based CPU with Linux. In other words, **adopting a non-x86 CPU and adopting commercial UNIX such as AIX are not necessarily a set.** ([TOP500][8])

In summary, in the past, **the value of manufacturers providing a complete, integrated computing system, including software, was significant,** and later, **the value of combining widely available components with a common OS and software became significant.**

It's not that "RISC and AIX were wrong choices," but rather that **the optimal configuration for performing the necessary calculations quickly and reliably for the same budget has changed.**
