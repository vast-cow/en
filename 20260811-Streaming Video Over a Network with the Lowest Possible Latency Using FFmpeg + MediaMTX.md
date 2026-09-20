---
title: "Streaming Video Over a Network with the Lowest Possible Latency Using FFmpeg + MediaMTX"
description: "When you want to send video captured from an HDMI capture device or similar source to another PC on..."
pubDatetime: 2026-08-11T06:54:54.962Z
updatedDate: 2026-08-12T04:18:56.930Z
---

When you want to send video captured from an HDMI capture device or similar source to another PC on the same LAN, one issue that can be surprisingly significant is **video latency**.

If you simply encode the video as H.264 and stream it over the network, small amounts of buffering can occur at multiple stages, such as:

* Capture
* FFmpeg’s internal queues
* Encoding
* Multiplexing
* Network transport
* Player-side buffering
* Decoding and display

As a result, the final latency can range from several hundred milliseconds to several seconds.

This time, we will build a low-latency streaming setup that minimizes buffering as much as possible using the following configuration:

**V4L2 + ALSA → FFmpeg → MediaMTX → RTSP/UDP → ffplay**

In the current FFmpeg documentation, `nobuffer` is defined as an option for reducing latency caused by buffering during input analysis, while `low_delay` is defined as a flag that forces low-delay operation.

---

## Configuration

The setup used here looks like this:

```text
HDMI input
   ↓
Capture device
 /dev/video0
   ↓
FFmpeg
  ├─ Video input via V4L2
  ├─ Audio input via ALSA
  ├─ H.264 encoding with Intel QSV
  ↓
RTSP / UDP
   ↓
MediaMTX
   ↓
LAN
   ↓
ffplay
   ↓
Display
```

Rather than having MediaMTX encode the video itself, **we use it as an RTSP server that relays the stream**.

MediaMTX is a media server that supports publishing and reading real-time video and audio streams, including RTSP, and it can also launch external commands as hooks.

This time, we will use its `runOnDemand` feature.

---

# MediaMTX Configuration

The `mediamtx.yml` file is configured as follows:

```yaml
paths:
  hdmi:
    runOnDemand: >-
      ffmpeg -hide_banner -y
      -loglevel warning
      -fflags nobuffer
      -flags low_delay
      -init_hw_device qsv=qsv
      -filter_hw_device qsv
      -thread_queue_size 4
      -f alsa
      -ac 2
      -ar 48000
      -i hw:1,0
      -thread_queue_size 1
      -f v4l2
      -input_format yuyv422
      -video_size 1920x1080
      -framerate 30
      -i /dev/video0
      -map 1:v:0
      -map 0:a:0
      -vf 'hwupload=extra_hw_frames=0,vpp_qsv=format=nv12'
      -c:v h264_qsv
      -preset veryfast
      -global_quality:v 35
      -async_depth 1
      -bf 0
      -g 30
      -c:a libopus -b:a 192k
      -f rtsp
      -rtsp_transport udp
      -muxdelay 0
      rtsp://127.0.0.1:8554/hdmi
    runOnDemandRestart: yes
    runOnDemandStartTimeout: 10s
    runOnDemandCloseAfter: 5s
```

At first glance, this may seem like a lot of options, but from a low-latency perspective, it becomes easier to understand if you break them down into a few key areas.

---

# Start FFmpeg Only When Needed with `runOnDemand`

First, let’s look at the MediaMTX side.

```yaml
runOnDemand: >-
  ffmpeg ...
```

`runOnDemand` is a feature that starts an external command when a client accesses the corresponding path.

In other words, it works like this:

```text
Nobody is watching
↓
FFmpeg stopped

ffplay connects to /hdmi
↓
MediaMTX starts FFmpeg
↓
FFmpeg publishes to /hdmi
↓
Playback starts
```

The official MediaMTX documentation also describes `runOnDemand` as a mechanism that starts the specified command when a reader requests the path.

Because FFmpeg does not need to run continuously, this is convenient when you only want to use HDMI streaming when needed.

In addition, we use:

```yaml
runOnDemandRestart: yes
```

so that if FFmpeg exits for some reason, it will be restarted. The current MediaMTX configuration reference also defines this option as a setting that restarts the command after it exits.

---

# Minimize Input-Side Buffers

An important principle in low-latency streaming is:

**Avoid “buffering first, processing later.”**

For that reason, the FFmpeg command begins with:

```text
-fflags nobuffer
-flags low_delay
```

## `-fflags nobuffer`

```text
-fflags nobuffer
```

This setting reduces latency caused by buffering during input stream analysis. The official FFmpeg documentation also describes it as an option for reducing latency caused by buffering during initial input analysis.

For real-time input, it is important to configure the pipeline in the following direction:

```text
Avoid accumulating packets as much as possible
↓
Pass incoming data to the next processing stage immediately
```

## `-flags low_delay`

```text
-flags low_delay
```

As the name suggests, this flag configures codec processing for low latency. In FFmpeg, `low_delay` is defined as “Force low delay.”

---

# Capture Video and Audio from Separate Devices

In this setup, audio is captured through ALSA:

```text
-f alsa
-ac 2
-ar 48000
-i hw:1,0
```

while video is captured through V4L2:

```text
-f v4l2
-input_format yuyv422
-video_size 1920x1080
-framerate 30
-i /dev/video0
```

From FFmpeg’s perspective, the inputs are therefore:

```text
input 0 = ALSA
input 1 = V4L2
```

The output streams are explicitly selected with:

```text
-map 1:v:0
-map 0:a:0
```

In other words:

```text
Video → video 0 from input 1
Audio → audio 0 from input 0
```

---

# Keep `thread_queue_size` Small

For the inputs, we specify:

```text
-thread_queue_size 4
```

and:

```text
-thread_queue_size 1
```

`thread_queue_size` determines how many packets read from an input device or other source FFmpeg may retain in its internal queue. The official FFmpeg documentation describes it, for inputs, as the maximum number of queued packets when reading from a device or file.

If you increase the queue size, behavior tends to become:

```text
Processing stalls slightly
↓
Packets accumulate in the queue
↓
Frames are processed without being dropped
```

However, in low-latency applications, this can become a problem:

```text
Processing cannot keep up
↓
Old video accumulates in the queue
↓
Displayed video falls further and further behind real time
```

For that reason, we use very small values here.

The design philosophy is:

**Prioritize displaying video that is close to the current time over guaranteeing that no frame is ever dropped.**

This is a very important concept in low-latency streaming.

---

# Encode with Intel Quick Sync Video

If 1920×1080 30 fps video is encoded to H.264 in software, the CPU load itself can become a source of latency.

For that reason, we use Intel Quick Sync Video, commonly known as QSV.

```text
-init_hw_device qsv=qsv
-filter_hw_device qsv
```

Then the video is uploaded to the QSV device and converted to NV12 with:

```text
-vf 'hwupload=extra_hw_frames=0,vpp_qsv=format=nv12'
```

The encoder is:

```text
-c:v h264_qsv
```

This allows the Intel GPU’s hardware encoder to handle H.264 encoding.

---

# Reduce Encoder “Lookahead” as Well

This is where some of the most important low-latency settings appear.

```text
-preset veryfast
-global_quality:v 35
-async_depth 1
-bf 0
-g 30
```

## `-preset veryfast`

```text
-preset veryfast
```

QSV presets include:

```text
veryfast
faster
fast
medium
slow
slower
veryslow
```

In FFmpeg, the `veryfast` side prioritizes speed, while the `veryslow` side prioritizes quality.

For live streaming, rather than pushing encoding quality to the limit, we prioritize:

**Encoding each frame quickly and sending it to the network.**

That is why we choose `veryfast`.

---

# `async_depth 1` Is Important

```text
-async_depth 1
```

This setting also helps reduce latency.

QSV can improve throughput by processing multiple frames asynchronously.

However:

```text
Process multiple frames in parallel
```

also means that multiple frames may exist inside the encoder at the same time.

The FFmpeg QSV documentation defines `async_depth` as a parameter related to the number of asynchronous operations, and for the QSV decoder it explicitly notes that increasing the value also increases latency.

For that reason, we reduce it to:

```text
-async_depth 1
```

The goal is a simple pipeline:

```text
Frame input
↓
Encode
↓
Output immediately
```

We prioritize latency over throughput.

---

# Do Not Use B-Frames

For low-latency H.264, this setting is especially important:

```text
-bf 0
```

When B-frames are used, encoding and decoding a frame may require referencing frames that occur later in time.

Conceptually, the structure may look like:

```text
I P B B P
```

As a result, the encoder and decoder may need to reorder frames, which is unfavorable for low-latency use cases.

Therefore, we completely disable B-frames with:

```text
-bf 0
```

Compression efficiency is sacrificed to some extent, but this is easier to handle in real-time applications.

---

# Use a One-Second GOP

```text
-g 30
```

This is another key setting.

Here, we use:

```text
-framerate 30
```

so the frame rate is 30 fps.

Therefore:

```text
30 frames ÷ 30 fps = 1 second
```

which means the GOP is divided at roughly one-second intervals.

In FFmpeg, `-g` specifies the GOP size.

Shortening the GOP slightly reduces compression efficiency, but it makes the stream easier to handle in situations such as:

* Playback startup
* Recovery after packet loss
* Joining the stream mid-session

For low-latency applications, compression ratio is not the only concern. **It is also important to catch up to the current video quickly.**

---

# `global_quality:v 35`

```text
-global_quality:v 35
```

This sets the QSV quality level.

When `global_quality:v` is specified with `h264_qsv`, quality-based rate control such as ICQ may be used depending on the conditions. The FFmpeg QSV documentation states that the ICQ range is 1–51, with **1 being the highest quality**.

In other words:

```text
Smaller value
↓
Higher quality, larger data volume

Larger value
↓
Lower quality, smaller data volume
```

`35` is fairly compression-oriented, so you can adjust it while monitoring network bandwidth and image quality.

For example, if you want better quality, you could lower it to:

```text
-global_quality:v 28
```

and compare the result.

---

# Audio Is Opus at 192 kbps

Audio is configured as:

```text
-c:a libopus -b:a 192k
```

Compared with video, audio encoding has relatively low computational and bandwidth requirements, so we use Opus at 192 kbps.

However, for monitoring applications where you only need to see the video, you can remove audio with:

```text
-an
```

to simplify the setup further.

---

# Use UDP for RTSP

FFmpeg outputs to MediaMTX with:

```text
-f rtsp
-rtsp_transport udp
-muxdelay 0
rtsp://127.0.0.1:8554/hdmi
```

For reducing latency, the key setting is:

```text
-rtsp_transport udp
```

FFmpeg allows UDP, TCP, and other methods to be selected as the RTSP lower transport. With UDP, media is sent over UDP; with TCP, the data is interleaved inside the RTSP control channel.

Unlike TCP, UDP does not have a mechanism like:

```text
Packet loss
↓
Wait for retransmission
↓
Subsequent processing also waits until the packet arrives
```

Therefore, it is well suited to real-time applications where the priority is:

**Even if the image becomes slightly corrupted, do not wait for old video—show the current video instead.**

Naturally, however, video may become corrupted in environments where packet loss is more likely, such as:

* Wi-Fi
* Congested LANs
* Streaming over the internet

If reliability is more important, another option is:

```text
-rtsp_transport tcp
```

Low latency and transmission reliability are a tradeoff.

---

# `muxdelay 0`

We also specify:

```text
-muxdelay 0
```

`muxdelay` is a delay-related parameter used on FFmpeg’s output side. The official FFmpeg RTSP sending example also includes a case using `-muxdelay 0.1`.

Here, we push it even further with:

```text
-muxdelay 0
```

to configure the system so that:

**As little time as possible is spent waiting to batch packets together.**

---

# Do Not Make MediaMTX Encode the Video

An important part of this setup is that FFmpeg publishes once to the local MediaMTX instance:

```text
FFmpeg
↓
rtsp://127.0.0.1:8554/hdmi
```

Clients then access:

```text
rtsp://x.x.x.x:8554/hdmi
```

In other words, MediaMTX acts as the relay point:

```text
FFmpeg
   ↓
MediaMTX
   ↓
Multiple clients
```

With this configuration, individual clients do not need direct access to the capture device or encoder.

It is also easier to manage when distributing the stream to multiple devices.

---

# Reduce Buffering on the ffplay Side as Well

Even if the sender is optimized for low latency, there is little benefit if the receiving player buffers one second of video before starting playback.

For that reason, ffplay is also configured as follows:

```bash
ffplay \
  -fflags nobuffer \
  -flags low_delay \
  -framedrop \
  -rtsp_transport udp \
  rtsp://x.x.x.x:8554/hdmi
```

---

## `-fflags nobuffer`

As on the sending side, we specify:

```text
-fflags nobuffer
```

to reduce latency caused by buffering during input analysis.

---

## `-flags low_delay`

```text
-flags low_delay
```

This also configures decoding for low-latency operation.

---

## `-framedrop`

```text
-framedrop
```

This is another important option for real-time applications.

ffplay provides the `framedrop` option, which allows video frames to be dropped when playback falls behind synchronization.

For low-latency use, instead of:

```text
Faithfully displaying every single frame
```

it is more important to:

```text
If processing falls behind, discard old frames
↓
Catch up to the current time
```

This setting sacrifices some visual continuity in order to prevent latency from increasing.

---

# In Low-Latency Streaming, “Dropping” Is Important

Looking at all of these settings, there is a common philosophy:

**Do not retain old data any longer than necessary.**

In normal video playback, it is important to:

```text
Avoid packet loss
Avoid dropping frames
Keep playback smooth
```

However, real-time video has different priorities.

For example, when displaying a game screen or camera feed, it may be more useful to show:

```text
Video from around 100 ms ago, even if some frames are skipped
```

than to perfectly display:

```text
Video from 3 seconds ago
```

For that reason, this setup reduces waiting time at every stage of the pipeline:

```text
Small input queues
↓
No B-frames
↓
Reduced QSV asynchronous depth
↓
Reduced muxer waiting time
↓
UDP
↓
Reduced buffering in ffplay
↓
Drop frames if playback falls behind
```

---

# Where Does Latency Occur?

When optimizing for low latency, you need to look at the entire pipeline, not just the network:

```text
Capture
↓
Queue
↓
Filter
↓
Encode
↓
Mux
↓
Network
↓
Demux
↓
Decode
↓
Render
```

For example, even if LAN ping is:

```text
1 ms
```

if you have:

```text
Encoder: 100 ms
Player buffer: 500 ms
```

then improving the network further will not have much impact.

In fact, with real-time video:

**Buffers inside the encoder, decoder, and player can sometimes contribute more latency than the network itself.**

That is why this configuration uses so many low-latency options.

---

# If You Want to Reduce Latency Even Further

If latency is still noticeable with this configuration, there are several additional things you can try.

First, ffplay’s RTSP reception uses a buffer for reordering UDP packets. The FFmpeg documentation states that packet reordering during UDP reception can be disabled by setting `max_delay` to 0.

For example, you can try:

```bash
ffplay \
  -fflags nobuffer \
  -flags low_delay \
  -framedrop \
  -rtsp_transport udp \
  -max_delay 0 \
  rtsp://x.x.x.x:8554/hdmi
```

However, this further reduces tolerance for out-of-order packets, so video may become unstable depending on network quality.

You can also reduce:

```text
-g 30
```

to something like:

```text
-g 15
```

However, shorter GOPs generally reduce compression efficiency.

Low-latency optimization always involves tradeoffs among:

```text
Latency
Image quality
Bandwidth
Stability
CPU/GPU load
```

---

# UDP Does Not Automatically Mean Low Latency

One important point to keep in mind is that:

```text
UDP = always low latency
```

is not necessarily true.

UDP can help avoid retransmission delays, but if network quality is poor, you may instead see:

```text
Packet loss
↓
Video corruption
↓
Wait until the next recoverable frame
```

Therefore, the ideal environment is something like:

**Wired LAN + UDP**

where packet loss is low and the network is stable.

In a Wi-Fi environment, it is better to test both UDP and TCP and compare:

```text
Actual latency
Video corruption
Stability
```

---

# Summary

In this setup, we built the following pipeline:

```text
HDMI
↓
V4L2 / ALSA
↓
FFmpeg
↓
Intel QSV H.264
↓
RTSP / UDP
↓
MediaMTX
↓
LAN
↓
ffplay
```

The key to reducing latency is not simply using a fast encoder.

What matters is removing, as much as possible, the buffers at each stage that exist to:

**“Hold a little data just in case.”**

The main settings used here can be summarized as follows:

```text
-fflags nobuffer
    Reduce input-side buffering

-flags low_delay
    Configure codecs for low latency

-thread_queue_size
    Keep capture input queues small

-c:v h264_qsv
    Encode H.264 quickly with Intel QSV

-preset veryfast
    Prioritize encoding speed

-async_depth 1
    Reduce the asynchronous processing depth inside QSV

-bf 0
    Disable B-frames

-g 30
    About a one-second GOP at 30 fps

-rtsp_transport udp
    Favor avoiding retransmission delays

-muxdelay 0
    Reduce waiting time in the muxer

-framedrop
    Drop old frames if playback falls behind
```

The important thing to understand about low-latency video streaming is that:

**“Deliver every frame reliably” and “deliver the current video” are different goals.**

For recording, not losing frames is important.

On the other hand, for real-time monitoring, remote control, game screens, camera surveillance, and similar applications, it is often more important to display the latest video even if a few frames are missing than to receive old video perfectly.

By tuning the capture, encoding, network, and player stages around that principle, you can achieve very low-latency video transmission even with RTSP.
