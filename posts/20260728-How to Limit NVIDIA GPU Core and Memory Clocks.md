---
title: "How to Limit NVIDIA GPU Core and Memory Clocks"
description: "Inspect supported NVIDIA clock ranges, apply and reset core or memory limits with nvidia-smi, and use power limits when direct clock control is unavailable."
pubDatetime: 2026-07-28T05:58:44.003Z
---

## Linux: Limit Clocks with `nvidia-smi`

On supported GPUs, you can directly specify the core clock and memory clock ranges. `root` privileges are required. GPU clock locking is supported on Volta-generation GPUs and later, but depending on the GPU model and driver, you may receive a `Not Supported` message. ([NVIDIA Docs][1])

### 1. Check the GPU and Supported Clocks

```bash
nvidia-smi -L
sudo nvidia-smi -q -d SUPPORTED_CLOCKS
sudo nvidia-smi -lmi
```

### 2. Limit the Core Clock

The following example limits GPU 0 to a minimum of 300 MHz and a maximum of 1500 MHz.

```bash
sudo nvidia-smi -i 0 --lock-gpu-clocks=300,1500
```

To lock it at exactly 1500 MHz:

```bash
sudo nvidia-smi -i 0 --lock-gpu-clocks=1500
```

Specifying a single value locks the GPU to that frequency, making it less likely to downclock while idle. In most cases, specifying a range as `min,max` is recommended. ([NVIDIA Docs][1])

### 3. Limit the Memory Clock

Example: set the memory clock range from 810 MHz to 5000 MHz.

```bash
sudo nvidia-smi -i 0 --lock-memory-clocks=810,5000
```

Lock the memory clock at 5000 MHz:

```bash
sudo nvidia-smi -i 0 --lock-memory-clocks=5000
```

Memory clock control is not supported on all GPUs. In addition, Hopper-based GPUs require the deferred-application method instead of the standard `--lock-memory-clocks` option. ([NVIDIA Docs][1])

### 4. Monitor the Current State

```bash
watch -n 1 'nvidia-smi --query-gpu=index,name,clocks.gr,clocks.mem,power.draw,temperature.gpu --format=csv'
```

### 5. Restore the Default Settings

```bash
sudo nvidia-smi -i 0 --reset-gpu-clocks
sudo nvidia-smi -i 0 --reset-memory-clocks
```

### If Clock Control Is Not Supported

You can indirectly limit the boost clock by reducing the GPU's power limit.

```bash
nvidia-smi -q -d POWER
sudo nvidia-smi -i 0 --power-limit=150
```

The specified value must fall within the displayed `Min Power Limit` and `Max Power Limit` range. ([NVIDIA Docs][1])

[1]: https://docs.nvidia.com/deploy/nvidia-smi/index.html "docs.nvidia.com"
[2]: https://jp.msi.com/support/technical_details/VGA_MSI_Utility_AfterBurner "How to Use MSI Afterburner"
[3]: https://jp.msi.com/blog/msi-afterburner-overclocking-undervolting-guide?utm_source=chatgpt.com "MSI Afterburner Walkthrough Part 1: Overclocking Guide & ..."
