---
title: "Setting Up and Troubleshooting FFmpeg + Intel QSV on Rocky Linux 9"
description: "If you want to accelerate FFmpeg H.264 / HEVC encoding using the integrated GPU in an Intel CPU, you..."
pubDatetime: 2026-08-07T08:31:28.851Z
---

If you want to accelerate FFmpeg H.264 / HEVC encoding using the integrated GPU in an Intel CPU, you can use Intel Quick Sync Video (QSV).

However, on Linux—especially on RHEL-based distributions such as Rocky Linux 9—simply seeing:

```text
FFmpeg recognizes h264_qsv
```

does not mean QSV will actually work.

In fact, on Rocky Linux 9, I encountered errors like the following:

```text
[AVHWDeviceContext @ 0x562042595380] Failed to initialise VAAPI connection: -1 (unknown libva error).
[h264_qsv @ 0x56204258e140] Failed to create a VAAPI device.
Error initializing output stream 0:0
```

Even when explicitly specifying the Intel GPU with `-qsv_device`, I got:

```text
[AVHWDeviceContext @ 0x55ac9a494500] Failed to initialise VAAPI connection: -1 (unknown libva error).
Device creation failed: -5.
Failed to set value '/dev/dri/renderD128' for option 'qsv_device': Input/output error
```

This article explains how to set up FFmpeg + QSV on Rocky Linux 9 and the order in which these types of errors should be isolated and diagnosed.

---

## Understanding the Layers Required for QSV to Work

The first important point is that QSV is not a standalone FFmpeg feature.

On Linux, access to the Intel GPU conceptually passes through several layers:

```text
FFmpeg
↓
QSV
↓
Intel Media SDK / oneVPL
↓
Intel Media Driver
↓
VA-API / libva
↓
/dev/dri/renderD128
↓
i915 / xe
↓
Intel GPU
```

Therefore, even if:

```bash
ffmpeg -encoders | grep qsv
```

shows:

```text
h264_qsv
hevc_qsv
```

that alone does not mean the GPU is actually usable.

For example, QSV will not work if there is a problem at any one of these points:

- Linux does not detect the Intel GPU
- `nomodeset` is configured
- `i915` / `xe` is not loaded
- `/dev/dri/renderD128` cannot be accessed
- `libva` is missing
- Intel Media Driver is missing or cannot be loaded
- The Media SDK / oneVPL runtime does not match
- FFmpeg itself was not built with QSV support

Therefore, the basic troubleshooting approach is to verify each layer from the bottom up.

---

# 1. Check Whether Linux Detects the Intel GPU

First, check the PCI devices.

```bash
lspci -nn | grep -Ei 'VGA|Display'
```

In this environment, the result was:

```text
00:02.0 VGA compatible controller [0300]:
Intel Corporation GeminiLake [UHD Graphics 605] [8086:3184] (rev 03)
```

This confirms that the Intel UHD Graphics 605, i.e. the Gemini Lake GPU, is detected.

Next, check the kernel driver.

```bash
lspci -nnk | grep -A4 -Ei 'VGA|Display'
```

The result in this case was:

```text
00:02.0 VGA compatible controller [0300]: Intel Corporation GeminiLake [UHD Graphics 605] [8086:3184] (rev 03)
DeviceName: Onboard - Video
Subsystem: Elitegroup Computer Systems Device [1019:a94d]
Kernel driver in use: i915
Kernel modules: i915
```

The important line here is:

```text
Kernel driver in use: i915
```

Gemini Lake uses `i915`.

On newer Intel GPUs, `xe` may be used depending on the configuration.

You can also verify this with:

```bash
lsmod | grep -E 'i915|xe'
```

---

# 2. Remove `nomodeset` If It Is Configured

When using an Intel iGPU with QSV / VA-API, specifying `nomodeset` in the kernel boot options can prevent the GPU driver from initializing correctly and can make QSV unusable.

First, check the current kernel command line.

```bash
cat /proc/cmdline
```

If it contains:

```text
nomodeset
```

remove it.

`nomodeset` disables Kernel Mode Setting (KMS). Because it interferes with the normal initialization of DRM/KMS drivers such as `i915`, which are used with Intel GPUs, it can prevent `/dev/dri/renderD128` from being created or cause VA-API initialization to fail even if a GPU device appears to exist.

On Rocky Linux 9, check the GRUB configuration.

```bash
sudo grubby --info=ALL | grep args
```

If `nomodeset` is configured, you can remove it from all kernel entries.

```bash
sudo grubby --update-kernel=ALL --remove-args="nomodeset"
```

After changing the setting, reboot.

```bash
sudo reboot
```

After rebooting, check again.

```bash
cat /proc/cmdline
```

After confirming that `nomodeset` is gone, run:

```bash
lspci -nnk | grep -A4 -Ei 'VGA|Display'
```

and confirm that it shows:

```text
Kernel driver in use: i915
```

Then check the DRM devices as well.

```bash
ls -l /dev/dri/
```

At a minimum, confirm that entries such as:

```text
card0
renderD128
```

have been created.

---

# 3. Check `/dev/dri/renderD128`

Next, check the DRM devices.

```bash
ls -l /dev/dri/
```

In this environment, the output was:

```text
drwxr-xr-x. 2 root root         80 Aug  7 16:04 by-path
crw-rw----. 1 root video  226,   0 Aug  7 16:04 card0
crw-rw-rw-. 1 root render 226, 128 Aug  7 16:04 renderD128
```

For server-side QSV and VA-API usage, the especially important device is:

```text
/dev/dri/renderD128
```

Even on a server that is not running X11 or Wayland, hardware encoding is possible as long as `renderD128` is accessible.

In other words:

```text
No GUI = QSV cannot be used
```

is not true.

QSV can also be used on headless servers.

---

# 4. Check Permissions on renderD128

A typical device node looks like this:

```text
crw-rw---- 1 root render ... /dev/dri/renderD128
```

In that case, add the user running FFmpeg to the `render` group.

```bash
sudo usermod -aG render $USER
```

Depending on the environment, membership in `video` may also be required, so this is also acceptable:

```bash
sudo usermod -aG video,render $USER
```

After logging in again, verify with:

```bash
id
```

If FFmpeg is launched from a systemd service, the required permissions must be granted not to the login user, but to **the user running the service**.

The same point matters when using Jellyfin, Plex, or custom transcoding workflows combined with MediaMTX.

In this environment, the device permissions were:

```text
crw-rw-rw-. 1 root render ... renderD128
```

so a simple Unix permission problem was unlikely.

---

# 5. Enable EPEL and RPM Fusion

The standard Rocky Linux 9 repositories may not contain all packages needed for FFmpeg and the Intel Media Driver stack.

First, add EPEL.

```bash
sudo dnf install -y epel-release
```

Then add RPM Fusion Free / Nonfree.

```bash
sudo dnf install -y \
https://download1.rpmfusion.org/free/el/rpmfusion-free-release-9.noarch.rpm \
https://download1.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-9.noarch.rpm
```

Then update the system.

```bash
sudo dnf update -y
```

---

# 6. Install Intel Media Driver

For relatively recent Intel GPUs, Intel Media Driver—the `iHD` driver—is used as the VA-API driver.

With Rocky Linux 9 + RPM Fusion, install:

```bash
sudo dnf install -y intel-media-driver
```

Intel Media Driver provides the VA-API backend:

```text
/usr/lib64/dri/iHD_drv_video.so
```

The Intel UHD Graphics 605 / Gemini Lake used here is also supported by Intel Media Driver.

---

# 7. Install libva and vainfo

`vainfo` is extremely useful for verifying VA-API operation.

```bash
sudo dnf install -y libva libva-utils
```

After installation, run:

```bash
vainfo
```

However, on a server without a GUI, it is more reliable to specify the DRM device explicitly.

```bash
vainfo --display drm --device /dev/dri/renderD128
```

If everything is working correctly, the output should look roughly like this:

```text
libva info: VA-API version ...
libva info: Trying to open /usr/lib64/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_...
libva info: va_openDriver() returns 0
vainfo: Driver version: Intel iHD driver ...
```

---

# 8. Explicitly Set `LIBVA_DRIVER_NAME=iHD`

If automatic detection does not work correctly, you can explicitly specify the VA-API driver to use.

```bash
LIBVA_DRIVER_NAME=iHD \
vainfo --display drm --device /dev/dri/renderD128
```

If you want to set it persistently, you can also use:

```bash
export LIBVA_DRIVER_NAME=iHD
```

---

# 9. Intel Media SDK vs. oneVPL

Several names appear around QSV:

```text
Intel Media SDK
libmfx
oneVPL
libvpl
intel-vpl-gpu-rt
```

Intel's newer software stack has transitioned from the legacy Intel Media SDK to oneVPL.

However, when deciding what to install for FFmpeg on Rocky Linux 9, it is important to **check which API stack your FFmpeg build was compiled against**.

For the RPM Fusion FFmpeg 5.1.10 used here:

```bash
ffmpeg -version
```

showed that it had been built with:

```text
--enable-libmfx
```

And when checking:

```bash
rpm -qa | grep -Ei 'libva|intel-media|libmfx|vpl'
```

the result was:

```text
libva-2.22.0-1.el9.x86_64
intel-mediasdk-21.3.5-1.el9.x86_64
libva-utils-2.11.1-1.el9.x86_64
```

In this environment, FFmpeg uses QSV through `libmfx`, i.e. Intel Media SDK.

Therefore, it is safer **not to assume that "Rocky 9 always requires libvpl + intel-vpl-gpu-rt."**

First, check the configure options shown by:

```bash
ffmpeg -version
```

---

# 10. Install FFmpeg

Install FFmpeg from RPM Fusion.

```bash
sudo dnf install -y ffmpeg
```

Verify it.

```bash
ffmpeg -version
```

---

# 11. Check Whether FFmpeg Supports QSV

First, check the list of hardware acceleration methods.

```bash
ffmpeg -hwaccels
```

For the FFmpeg build used here, the result was:

```text
Hardware acceleration methods:
vdpau
cuda
vaapi
qsv
drm
opencl
vulkan
```

Here, you can confirm:

```text
vaapi
qsv
```

Next, check the QSV encoders.

```bash
ffmpeg -hide_banner -encoders | grep -E 'qsv|vaapi'
```

In this environment, the output was:

```text
V..... h264_qsv     H.264 / AVC ... (Intel Quick Sync Video acceleration)
V....D h264_vaapi   H.264/AVC (VAAPI)
V..... hevc_qsv     HEVC (Intel Quick Sync Video acceleration)
V....D hevc_vaapi   H.265/HEVC (VAAPI)
V..... mjpeg_qsv    MJPEG (Intel Quick Sync Video acceleration)
V....D mjpeg_vaapi  MJPEG (VAAPI)
V..... mpeg2_qsv    MPEG-2 video (Intel Quick Sync Video acceleration)
V....D mpeg2_vaapi  MPEG-2 (VAAPI)
V....D vp8_vaapi    VP8 (VAAPI)
V....D vp9_vaapi    VP9 (VAAPI)
V..... vp9_qsv      VP9 video (Intel Quick Sync Video acceleration)
```

**Seeing `h264_qsv` in the list and actually being able to use the GPU are two different things.**

---

# 12. Get `vainfo` Working First

When troubleshooting QSV, it is usually faster to get:

```bash
vainfo --display drm --device /dev/dri/renderD128
```

working first rather than repeatedly changing FFmpeg options.

In this environment, it returned:

```text
libva info: VA-API version 1.22.0
libva info: Trying to open /usr/lib64/dri/iHD_drv_video.so
libva info: va_openDriver() returns -1
libva info: Trying to open /usr/lib64/dri/i965_drv_video.so
libva info: va_openDriver() returns -1
vaInitialize failed with error code -1 (unknown libva error),exit
```

Because the GPU was detected, `i915` was in use, and `/dev/dri/renderD128` existed, this narrowed the problem down to the `libva` / Intel Media Driver area.

---

# 13. Check Which RPM Provides `iHD_drv_video.so`

```bash
ls -l /usr/lib64/dri/iHD_drv_video.so
```

Then use:

```bash
rpm -qf /usr/lib64/dri/iHD_drv_video.so
```

to check which RPM provides the file.

If you do not know which RPM contains it, you can also search with:

```bash
dnf provides '*/iHD_drv_video.so'
```

The expected provider is an `intel-media-driver` package.

```bash
sudo dnf install -y intel-media-driver
```

One important point is that `intel-mediasdk` and `intel-media-driver` are different packages.

---

# 14. Check the Driver's Library Dependencies

```bash
ldd /usr/lib64/dri/iHD_drv_video.so
```

If this output contains:

```text
not found
```

then a required dependency is missing.

---

# 15. Check Package Sources and Versions

```bash
rpm -qi libva libva-utils intel-mediasdk intel-media-driver
```

Or use:

```bash
dnf repoquery --installed \
--qf '%{name} %{version}-%{release} %{repoid}' \
libva libva-utils intel-mediasdk intel-media-driver
```

---

# 16. Explicitly Test with iHD

```bash
LIBVA_DRIVER_NAME=iHD \
vainfo --display drm --device /dev/dri/renderD128
```

If it is working correctly, the output should include:

```text
Trying to open /usr/lib64/dri/iHD_drv_video.so
Found init function ...
va_openDriver() returns 0
```

---

# 17. Run a Standalone QSV Test

Once `vainfo` works correctly, test QSV encoding using a generated test pattern that is unrelated to any input video.

```bash
ffmpeg \
-qsv_device /dev/dri/renderD128 \
-f lavfi \
-i testsrc2=size=1280x720:rate=30 \
-t 5 \
-c:v h264_qsv \
-global_quality 23 \
-f null -
```

If this succeeds, you can conclude that the H.264 QSV encoding path is functioning.

---

# 18. Isolate the Problem with VAAPI Encoding

```bash
ffmpeg \
-vaapi_device /dev/dri/renderD128 \
-f lavfi \
-i testsrc2=size=1280x720:rate=30 \
-vf 'format=nv12,hwupload' \
-t 5 \
-c:v h264_vaapi \
-f null -
```

| VAAPI | QSV | Possible Cause |
|---|---|---|
| NG | NG | VA-API / Intel Media Driver / GPU device side |
| OK | NG | Media SDK / oneVPL / QSV runtime side |
| OK | OK | GPU stack is healthy. Investigate the original FFmpeg command |
| NG | OK | An unusual configuration that is normally uncommon |

---

# 19. Encode H.264 with QSV

```bash
ffmpeg \
-qsv_device /dev/dri/renderD128 \
-i input.mp4 \
-c:v h264_qsv \
-global_quality 23 \
-c:a copy \
output.mp4
```

---

# 20. Encode HEVC / H.265 with QSV

```bash
ffmpeg \
-qsv_device /dev/dri/renderD128 \
-i input.mp4 \
-c:v hevc_qsv \
-global_quality 25 \
-c:a copy \
output.mp4
```

---

# 21. QSV Decode + QSV Encode

```bash
ffmpeg \
-qsv_device /dev/dri/renderD128 \
-hwaccel qsv \
-hwaccel_output_format qsv \
-i input.mp4 \
-c:v h264_qsv \
-global_quality 23 \
-c:a copy \
output.mp4
```

For troubleshooting, it is easier to isolate problems by starting with CPU decode + QSV encode.

---

# 22. The `yuv420p` → `nv12` Warning Is Not a Fatal Error

```text
Incompatible pixel format 'yuv420p' for codec 'h264_qsv',
auto-selecting format 'nv12'
```

This is unrelated to the VA-API error discussed here.

If necessary, you can explicitly specify:

```bash
-pix_fmt nv12
```

---

# 23. ALSA `Thread message queue blocking` Is Also a Separate Issue

If you see:

```text
[alsa] Thread message queue blocking;
consider raising the thread_queue_size option
```

specify something like:

```bash
-thread_queue_size 1024
```

before the ALSA input.

---

# 24. Example Rocky Linux 9 Setup

```bash
sudo dnf install -y epel-release
sudo dnf install -y \
https://download1.rpmfusion.org/free/el/rpmfusion-free-release-9.noarch.rpm \
https://download1.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-9.noarch.rpm
sudo dnf install -y \
ffmpeg \
libva \
libva-utils \
intel-media-driver
```

In an environment like this one where `ffmpeg -version` includes:

```text
--enable-libmfx
```

this package is also a candidate:

```bash
sudo dnf install -y intel-mediasdk
```

For Gemini Lake + RPM Fusion FFmpeg 5.1.x, a straightforward starting configuration is roughly:

```bash
sudo dnf install -y \
ffmpeg \
libva \
libva-utils \
intel-media-driver \
intel-mediasdk
```

---

# 25. Final Verification Checklist

```bash
# 0. nomodeset
cat /proc/cmdline

# 1. GPU / kernel driver
lspci -nnk | grep -A4 -Ei 'VGA|Display'

# 2. Kernel module
lsmod | grep -E 'i915|xe'

# 3. DRM
ls -l /dev/dri/

# 4. Intel Media Driver
rpm -q intel-media-driver

# 5. iHD driver
ls -l /usr/lib64/dri/iHD_drv_video.so
rpm -qf /usr/lib64/dri/iHD_drv_video.so

# 6. VA-API
LIBVA_DRIVER_NAME=iHD \
vainfo --display drm --device /dev/dri/renderD128

# 7. FFmpeg HW acceleration
ffmpeg -hwaccels

# 8. QSV encoders
ffmpeg -hide_banner -encoders | grep _qsv

# 9. QSV decoders
ffmpeg -hide_banner -decoders | grep _qsv

# 10. QSV encode test
ffmpeg \
-qsv_device /dev/dri/renderD128 \
-f lavfi \
-i testsrc2=size=1280x720:rate=30 \
-t 5 \
-c:v h264_qsv \
-global_quality 23 \
-f null -
```

The verification order is easiest to understand as follows:

```text
Presence of nomodeset
↓
Intel GPU
↓
i915 / xe
↓
/dev/dri/renderD128
↓
VA-API / libva
↓
Intel Media Driver
↓
Media SDK / oneVPL
↓
FFmpeg
```

---

# What We Learned with Gemini Lake / UHD Graphics 605

On the actual system used here:

```text
Intel Corporation GeminiLake [UHD Graphics 605]
Kernel driver in use: i915
```

and `/dev/dri/renderD128` was also present.

In addition, FFmpeg had been built with `--enable-libmfx`, recognized `qsv` / `vaapi`, and could list encoders such as `h264_qsv`, `hevc_qsv`, and `vp9_qsv`.

However:

```bash
vainfo --display drm --device /dev/dri/renderD128
```

returned:

```text
Trying to open /usr/lib64/dri/iHD_drv_video.so
va_openDriver() returns -1
Trying to open /usr/lib64/dri/i965_drv_video.so
va_openDriver() returns -1
```

The QSV test also failed with:

```text
Failed to initialise VAAPI connection
Device creation failed
```

This shows that the problem is not with FFmpeg encoding options, but with the VA-API / Intel Media Driver layer.

In this situation, investigate the Intel Media Driver installation and dependencies in this order:

```bash
rpm -qf /usr/lib64/dri/iHD_drv_video.so
ldd /usr/lib64/dri/iHD_drv_video.so
LIBVA_DRIVER_NAME=iHD \
vainfo --display drm --device /dev/dri/renderD128
```

---

# Summary

When using FFmpeg + Intel QSV on Rocky Linux 9, seeing `h264_qsv` in `ffmpeg -encoders` does not mean the setup is complete.

The required layers are roughly as follows:

| Component | Requirement | Role |
|---|---|---|
| Disable `nomodeset` | Important | Allows DRM/KMS to initialize correctly |
| Intel GPU | Required | Hardware |
| `i915` / `xe` | Required | Kernel GPU driver |
| `/dev/dri/renderD128` | Required | DRM render node |
| `libva` | Important in Linux QSV environments | VA-API |
| `intel-media-driver` | Important for supported Intel GPUs | `iHD_drv_video.so` |
| `intel-mediasdk` | For libmfx-based setups | Legacy QSV runtime |
| `libvpl` | For oneVPL-based setups | oneVPL dispatcher |
| `intel-vpl-gpu-rt` | For supported newer-generation GPUs | oneVPL GPU implementation |
| QSV-enabled FFmpeg | Required | `h264_qsv` / `hevc_qsv`, etc. |

In particular, if you see:

```text
Failed to initialise VAAPI connection
Failed to create a VAAPI device
```

the fastest approach is to check:

```bash
cat /proc/cmdline
vainfo --display drm --device /dev/dri/renderD128
```

before changing encoding options.

If `nomodeset` is still present, remove it. If `vainfo` does not work correctly, fix the VA-API / Intel Media Driver problem first.

When building a QSV environment on Rocky Linux 9, keep the following layers in mind:

```text
nomodeset
↓
GPU
↓
Kernel driver
↓
DRM
↓
VA-API
↓
Intel Media Driver
↓
Media SDK / oneVPL
↓
FFmpeg
```

The most reliable setup method is to **verify proper operation one layer at a time, from the bottom up**.
