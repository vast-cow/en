---
title: "Tampermonkey Script to Fix the Issue Where Long Album Titles Disappear in YouTube Music"
description: "Correct YouTube Music album-title clipping with a userscript that moves two-line truncation onto the title element and adjusts link display."
pubDatetime: 2026-07-15T06:36:09.243Z
updatedDate: 2026-07-15T07:20:26.774Z
---

In YouTube Music, long album titles may sometimes disappear from the screen even though the title element itself still exists in the page.

This Tampermonkey script fixes the display issue by overriding the relevant CSS, allowing long album titles to be displayed correctly.

## What This Script Does

* Fixes the issue where long album titles disappear.
* Displays album titles on up to two lines, truncating any additional text.
* Preserves YouTube Music's original layout as much as possible.

The script only affects long titles, leaving normal titles unchanged.

## How to Use

1. Install Tampermonkey in your browser.
2. Create a new user script.
3. Paste the script into the editor and save it.
4. Reload YouTube Music.

If you want to apply the fix to local MHTML files, enable **"Allow access to file URLs"** in the Tampermonkey settings.

## How It Works

The script overrides the CSS responsible for rendering album titles and applies the two-line truncation directly to the title element itself.

As a result, it prevents long album titles from being pushed outside the visible area and becoming invisible.

## Summary

By installing this Tampermonkey script, you can easily fix the issue where long album titles disappear in YouTube Music.

Once installed, no additional configuration is required—the fix is applied automatically whenever you use YouTube Music.

```javascript
// ==UserScript==
// @name         YouTube Music Long Album Title Display Fix
// @namespace    local.youtube-music-fixes
// @version      1.0.0
// @description  Fixes an issue where long album titles move outside the overflow clipping area and become invisible.
// @match        https://music.youtube.com/*
// @match        file:///*
// @grant        GM_addStyle
// @run-at       document-start
// ==/UserScript==

(() => {
  'use strict';

  const css = `
    /*
     * Do not apply line clamping to the title-group itself.
     * Instead, apply the two-line clamp to the actual title element.
     */
    ytmusic-two-row-item-renderer
      .title-group.ytmusic-two-row-item-renderer {
      display: block !important;
      -webkit-line-clamp: unset !important;
      -webkit-box-orient: initial !important;
      overflow: visible !important;
      max-height: none !important;
    }

    /*
     * Clamp the title element itself to two lines.
     */
    ytmusic-two-row-item-renderer
      .title.ytmusic-two-row-item-renderer {
      display: -webkit-box !important;
      -webkit-box-orient: vertical !important;
      -webkit-line-clamp: 2 !important;

      overflow: hidden !important;
      white-space: normal !important;
      text-overflow: ellipsis !important;

      max-height: calc(
        var(--ytmusic-responsive-font-size, 14px)
        * 2
        * var(--ytmusic-title-line-height, 1.3)
      ) !important;
    }

    /*
     * Prevent the inline-block link from becoming a single multi-line box
     * that gets pushed outside the parent's clipping region.
     */
    ytmusic-two-row-item-renderer
      .title.ytmusic-two-row-item-renderer
      > a.yt-simple-endpoint.yt-formatted-string {
      display: inline !important;
    }
  `;

  function installStyle() {
    if (typeof GM_addStyle === 'function') {
      GM_addStyle(css);
      return;
    }

    const style = document.createElement('style');
    style.id = 'ytmusic-long-album-title-fix';
    style.textContent = css;

    const target = document.head || document.documentElement;
    target.appendChild(style);
  }

  installStyle();
})();
```
