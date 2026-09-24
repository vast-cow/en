---
title: "Using ffplay as a Lightweight Presentation Preview Window"
description: "Preview a Windows secondary or virtual display in an ffplay window using Desktop Duplication capture and low-latency settings for screen sharing."
pubDatetime: 2026-06-15T07:33:58.866Z
---

When giving a presentation over Zoom or a similar video conferencing tool, it is often useful to show the presentation screen inside a normal window. This makes it easier to check what is being presented, arrange windows on the desktop, and share only the intended content.

I used to do this with a virtual display driver and **raiton**. The virtual display acted as an additional monitor, and raiton displayed that monitor in a window. This setup worked, but after looking into it again, I found that **ffplay**, which is included with FFmpeg, can be a better alternative.

## Purpose

The goal is simple: display a secondary or virtual monitor inside a window.

This is useful when presenting slides in PowerPoint or similar software. For example, PowerPoint can show the presentation on a second display, while ffplay shows that display as a window on the main desktop. That window can then be monitored or shared in Zoom.

## Using ffplay Instead of raiton

The following command displays the second screen in a window:

```bash
ffplay -fflags nobuffer -flags low_delay -framedrop -f lavfi "ddagrab=output_idx=1:framerate=60,hwdownload,format=bgra,format=bgr0"
```

This uses FFmpeg’s `ddagrab` source to capture a desktop output and display it with ffplay.

In this command, `output_idx=1` means that the second display is captured. If the target display is different, this number may need to be changed.

The options `nobuffer`, `low_delay`, and `framedrop` help reduce preview latency, which is important when using the window during a live presentation.

## Performance

In my environment, raiton could display the virtual display at around 15 fps. With ffplay, I was able to display it at 60 fps.

This makes ffplay feel much smoother, especially when slides include animations, cursor movement, or other visual changes.

## Virtual Display Driver

For the virtual display, I use [ge9/IddSampleDriver](https://github.com/ge9/IddSampleDriver).

This provides an additional display output on Windows. Presentation software can use that virtual display as if it were a real monitor, and ffplay can then show that display in a normal window.

## Summary

ffplay can be used as a practical replacement for raiton when you want to preview or share a presentation display inside a window.

It requires only FFmpeg, works well with a virtual display driver, and can provide smoother output than raiton in this use case. For presentations over Zoom or similar tools, this is a simple and effective way to handle a separate presentation screen.
