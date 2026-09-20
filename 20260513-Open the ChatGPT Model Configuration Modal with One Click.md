---
title: "Open the ChatGPT Model Configuration Modal with One Click"
description: "This userscript automatically clicks specific elements related to the ChatGPT model configuration..."
pubDatetime: 2026-05-13T04:04:24.247Z
updatedDate: 2026-05-13T09:06:03.199Z
---

This userscript automatically clicks specific elements related to the ChatGPT model configuration modal when they appear on the page. It is intended for use with a userscript manager such as Tampermonkey.

## Purpose

The purpose of this script is to reduce the need for repeated manual clicks when a specific modal-related element appears while using ChatGPT.

The script looks for elements with the following attribute:

```javascript
data-testid="model-configure-modal"
```

When it finds matching elements, it automatically clicks them.

## What It Does

This script runs only on ChatGPT pages:

```javascript
https://chatgpt.com/*
```

When the page loads, it immediately checks whether the target element is already present. If it is found, the script clicks it automatically.

It also continues watching the page for changes. This means that even if the modal appears later, after the initial page load, the script can still detect it and click it.

## How to Use It

## 1. Install Tampermonkey

Install Tampermonkey, or another userscript manager, in your browser.

## 2. Create a New Userscript

Open the Tampermonkey dashboard and create a new userscript.

## 3. Paste the Script

Paste the full script into the editor.

The header at the top of the script defines its name, version, description, and the website where it should run. In this case, it is limited to ChatGPT pages.

## 4. Save the Script and Open ChatGPT

Save the script, then open:

```text
https://chatgpt.com/
```

When the target modal element appears, the script will click it automatically.

## Checking the Logs

The script prints messages to the browser console. These logs can help you confirm that the script is running and that it has found the target element.

Example log messages may include:

```text
Started.
Found 1 target(s).
Clicking target #1.
MutationObserver attached to document.body.
```

These messages show when the script starts, how many target elements it finds, and when it clicks them.

## Handling Page Changes

ChatGPT can update parts of the page without fully reloading it. Because of this, the target element may appear after the page has already loaded.

To handle that, the script uses a `MutationObserver`. This lets it watch for changes inside the page and run the click check again whenever new elements are added.

## Notes

This script depends on ChatGPT’s current page structure. If the target attribute or page layout changes, the script may stop working.

It is also important to understand what the script is clicking. Since it performs automatic clicks, it may cause unintended actions if the target selector matches something unexpected.

## Summary

This userscript is a simple automation tool for clicking a specific ChatGPT model configuration modal element.

It checks for the target element when the page loads and keeps watching for it afterward. This makes it useful when the same modal appears repeatedly or is inserted dynamically after the page has loaded.

```javascript
// ==UserScript==
// @name         ChatGPT Model Configure Modal Auto Clicker
// @namespace    http://tampermonkey.net/
// @version      1.3
// @description  Automatically clicks ChatGPT model configure modal elements when they appear.
// @match        https://chatgpt.com/*
// @grant        none
// ==/UserScript==

(function () {
  const selector = '*[data-testid="model-configure-modal"]';
  const intervalMs = 100;

  let lastExecutedAt = 0;
  let reservedTimer = null;

  const log = (...args) => {
    console.log('[bookmarklet:model-configure-modal]', ...args);
  };

  const clickIfFound = () => {
    lastExecutedAt = Date.now();
    reservedTimer = null;

    const targets = document.querySelectorAll(selector);

    log(`Found ${targets.length} target(s).`);

    targets.forEach((target, index) => {
      log(`Clicking target #${index + 1}.`, target);
      target.click();
    });
  };

  const requestClickIfFound = () => {
    const now = Date.now();
    const elapsed = now - lastExecutedAt;

    if (elapsed >= intervalMs) {
      clickIfFound();
      return;
    }

    if (reservedTimer !== null) {
      log('Execution already reserved. Skipped.');
      return;
    }

    const delay = intervalMs - elapsed;

    log(`Executed recently. Reserving execution in ${delay}ms.`);

    reservedTimer = setTimeout(() => {
      clickIfFound();
    }, delay);
  };

  log('Started.');

  requestClickIfFound();

  const observer = new MutationObserver((mutations) => {
    log(`Mutation observed. mutation count: ${mutations.length}`);

    requestClickIfFound();
  });

  observer.observe(document.body, {
    childList: true,
    subtree: true
  });

  log('MutationObserver attached to document.body.');
})();
```
