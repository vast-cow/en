---
title: "Which NVIDIA HPC SDK Versions Bundle (or Don’t Bundle) CUDA 12 and CUDA 13"
description: "Goal / Interpretation of “latest” Latest HPC SDK version that matches Bundled CUDA toolchains (as..."
pubDatetime: 2026-02-24T07:36:07.502Z
---

| Goal / Interpretation of “latest”                                            | Latest HPC SDK version that matches | Bundled CUDA toolchains (as stated in the sources) | Notes / Boundary                                                |
| ---------------------------------------------------------------------------- | ----------------------------------: | -------------------------------------------------- | --------------------------------------------------------------- |
| **Latest NVHPC that does *not* bundle CUDA 13.0**                            |                            **25.7** | **CUDA 11.8 + CUDA 12.9U1**                        | CUDA 13.0 bundling starts at **25.9**                           |
| **First NVHPC that bundles CUDA 13.0**                                       |                            **25.9** | **CUDA 13.0 + CUDA 12.9U1**                        | Marks the transition to CUDA 13.0 in the bundle                 |
| **Latest NVHPC that does *not* bundle CUDA 12** (i.e., last pre-CUDA-12 era) |                           **22.11** | **CUDA 10.2 / 11.0 / 11.8**                        | CUDA 12 bundling begins at **23.1**                             |
| **First NVHPC that bundles CUDA 12**                                         |                            **23.1** | **CUDA 12.0** (bundled/support introduced)         | From here onward, the “no CUDA 12 bundled” condition is not met |
