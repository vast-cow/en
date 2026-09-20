---
title: "Restoring Hidden Playback Buttons in YouTube Music"
description: "Overview   This user script is designed to fix an issue in YouTube Music where certain..."
pubDatetime: 2026-04-27T08:54:58.897Z
updatedDate: 2026-06-10T02:57:31.012Z
---

## Overview

This user script is designed to fix an issue in YouTube Music where certain playback controls—specifically the rewind and fast-forward buttons—may become hidden. It works by periodically scanning the page and restoring the visibility of these buttons when necessary.

## Purpose

In some cases, YouTube Music’s interface may hide key controls such as:

* The rewind (10 seconds back) button
* The fast-forward (30 seconds ahead) button

When these controls are not visible, it can disrupt normal playback navigation. This script ensures that those buttons remain accessible, improving overall usability.

## How to Use

### 1. Set Up a User Script Manager

To run this script, you need a browser extension that supports user scripts. A commonly used option is:

* Tampermonkey (available for Chrome, Edge, Firefox, and others)

### 2. Install the Script

1. Open the Tampermonkey dashboard
2. Create a new script
3. Paste the provided code into the editor and save it

### 3. Open YouTube Music

The script automatically runs when you visit:

* [https://music.youtube.com/](https://music.youtube.com/)

No additional configuration is required.

## How It Works (Briefly)

* The script looks for playback buttons associated with rewind and fast-forward actions
* If these elements have a `hidden` attribute, it removes it
* This process runs every 5 seconds to ensure the buttons stay visible

## Notes

* If YouTube Music updates its interface, the script may need adjustments
* The script runs periodically, but the performance impact is minimal

## Summary

This simple user script helps maintain access to essential playback controls in YouTube Music. By automatically restoring hidden buttons, it provides a smoother and more consistent listening experience.

```javascript
// ==UserScript==
// @name         YouTube Music Hidden Buttons Fix
// @namespace    http://tampermonkey.net/
// @version      1.2
// @description  Unhide rewind/forward buttons when the left control area changes
// @match        https://music.youtube.com/*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    const MIN_INTERVAL_MS = 1000;

    let lastRunAt = 0;
    let scheduledTimer = null;

    function unhideButtons() {
        // yt-icon-button elements that contain a replay_10 icon
        document.querySelectorAll(
            'yt-icon-button:has(yt-icon[icon="yt-sys-icons\\:replay_10"])'
        ).forEach(el => {
            el.removeAttribute("hidden");
        });

        // yt-icon-button elements that contain a skip_forward_30 icon
        document.querySelectorAll(
            'yt-icon-button:has(yt-icon[icon="yt-sys-icons\\:skip_forward_30"])'
        ).forEach(el => {
            el.removeAttribute("hidden");
        });
    }

    function runThrottled() {
        const now = Date.now();
        const elapsed = now - lastRunAt;

        // Execute immediately if at least 1000 ms have passed since the previous run
        if (elapsed >= MIN_INTERVAL_MS) {
            if (scheduledTimer !== null) {
                clearTimeout(scheduledTimer);
                scheduledTimer = null;
            }

            lastRunAt = now;
            unhideButtons();
            return;
        }

        // Do not schedule another execution if one is already pending
        if (scheduledTimer !== null) {
            return;
        }

        // Execute after the remaining time needed to satisfy the 1000 ms interval
        const delay = MIN_INTERVAL_MS - elapsed;

        scheduledTimer = setTimeout(() => {
            scheduledTimer = null;
            lastRunAt = Date.now();
            unhideButtons();
        }, delay);
    }

    function observeLeftControls() {
        const target = document.getElementById("left-controls");

        if (!target) {
            // Retry because YouTube Music may not have finished rendering yet
            setTimeout(observeLeftControls, 500);
            return;
        }

        const observer = new MutationObserver(() => {
            runThrottled();
        });

        observer.observe(target, {
            childList: true,
            subtree: true,
            attributes: true,
            attributeFilter: ["hidden", "style", "class"]
        });

        // Run once after setting up the observer
        runThrottled();
    }

    // Initial execution
    runThrottled();

    // Observe left-controls instead of polling every 5 seconds
    observeLeftControls();
})();
```
