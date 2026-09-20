---
title: "(Out of order) A Tampermonkey Script That Adds a Model Settings Button to ChatGPT"
description: "This Tampermonkey script is a simple way to make ChatGPT easier to use by adding a Config button to..."
pubDatetime: 2026-04-20T08:06:24.080Z
updatedDate: 2026-05-13T03:44:06.754Z
---

This Tampermonkey script is a simple way to make ChatGPT easier to use by adding a **Config** button to the page. With that button, you can open the model settings screen more quickly without going through the usual steps in the interface.

## Purpose

The main purpose of this script is to improve convenience. If you often open the model settings in ChatGPT, repeating the same clicks can become tedious. This script shortens that process by placing a dedicated button in the page header.

Instead of searching through the interface each time, you can use a single visible entry point. This makes the settings screen easier to access during regular use.

## How It Works

The script runs on `https://chatgpt.com/*` and waits for the page header to become available. Once the correct part of the header appears, it inserts a new **Config** button there.

When you click that button, the script uses the existing elements already present in ChatGPT’s interface. It first activates the model switcher button, then attempts to open the configuration modal. In other words, it does not create a new settings system of its own. It simply provides a faster way to reach the one already built into ChatGPT.

## How to Use It

### Install the script in Tampermonkey

First, make sure Tampermonkey is installed in your browser. Then create a new user script, paste the provided code into it, and save it.

### Open ChatGPT

Go to ChatGPT in your browser. The script will automatically run on matching pages.

### Click the Config button

After the page loads, a **Config** button should appear in the header area. Clicking it will trigger the actions needed to open the model settings screen.

## Why It Is Useful

This script is useful because it reduces friction in everyday use. For people who frequently adjust model-related settings, even a small shortcut can make the interface feel smoother and more efficient.

It also includes a waiting mechanism so it does not try to add the button before the necessary page elements are ready. That helps it work more reliably on pages that load dynamically.

## Things to Keep in Mind

This script depends on ChatGPT’s current page structure and element names. If the site layout or internal identifiers change, the script may stop working correctly or may need to be updated.

It is also important to remember that this is a browser-side customization, not an official ChatGPT feature. Its behavior may vary depending on browser setup or future changes to the site.

## Conclusion

This Tampermonkey script is a practical customization for users who want quicker access to ChatGPT’s model settings screen. By adding a simple **Config** button to the header, it makes a common action faster and more direct.

It is a small change, but one that can improve the overall experience for people who use ChatGPT regularly.

```javascript
// ==UserScript==
// @name         Add Config Button With page-header Reattach Detection
// @namespace    http://tampermonkey.net/
// @version      1.2
// @description  Wait for #page-header, add Config button, and re-add it when #page-header is recreated
// @match        https://chatgpt.com/*
// @grant        none
// ==/UserScript==

(function () {
  'use strict';

  const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

  const BUTTON_ID = 'tm-config-button';
  let headerRemovalObserver = null;
  let isRunning = false;

  async function waitForPageHeader(timeoutMs = 30000, intervalMs = 100) {
    const start = Date.now();

    while (Date.now() - start < timeoutMs) {
      const pageHeader = document.querySelector('#page-header');
      if (pageHeader) {
        return pageHeader;
      }
      await sleep(intervalMs);
    }

    throw new Error('Timeout: #page-header was not found.');
  }

  function findButtonContainer(pageHeader) {
    return pageHeader.childNodes[1] || null;
  }

  async function onConfigClick(e) {
    console.log('Config button clicked', e);

    const switcherButton = document.querySelector('button[data-testid="model-switcher-dropdown-button"]');
    if (!switcherButton) {
      console.warn('model-switcher-dropdown-button not found');
      return;
    }

    switcherButton.click();

    await sleep(0);

    const configureModal = document.querySelector('div[data-testid="model-configure-modal"]');
    if (!configureModal) {
      console.warn('model-configure-modal not found');
      return;
    }

    configureModal.click();
  }

  function createConfigButton() {
    const button = document.createElement('button');
    button.id = BUTTON_ID;
    button.innerText = 'Config';
    button.className =
      'group flex cursor-pointer justify-center items-center gap-1 rounded-lg min-h-9 touch:min-h-10 px-2.5 text-lg hover:bg-token-surface-hover focus-visible:bg-token-surface-hover font-normal whitespace-nowrap focus-visible:outline-none';
    button.addEventListener('click', onConfigClick);
    return button;
  }

  function addConfigButton(pageHeader) {
    const target = findButtonContainer(pageHeader);
    if (!target) {
      console.warn('#page-header.childNodes[1] not found');
      return false;
    }

    if (target.querySelector(`#${BUTTON_ID}`)) {
      return true;
    }

    const configButton = createConfigButton();
    target.appendChild(configButton);
    console.log('Config button added');

    return true;
  }

  function observePageHeaderRemoval(pageHeader) {
    if (headerRemovalObserver) {
      headerRemovalObserver.disconnect();
      headerRemovalObserver = null;
    }

    headerRemovalObserver = new MutationObserver(() => {
      if (!document.contains(pageHeader)) {
        console.log('#page-header was removed. Restarting...');
        headerRemovalObserver.disconnect();
        headerRemovalObserver = null;
        run();
      }
    });

    const root = document.body || document.documentElement;
    headerRemovalObserver.observe(root, {
      childList: true,
      subtree: true,
    });
  }

  async function run() {
    if (isRunning) return;
    isRunning = true;

    try {
      // (0) Wait for #page-header
      const pageHeader = await waitForPageHeader();

      // (1) Add config button
      const added = addConfigButton(pageHeader);
      if (!added) {
        isRunning = false;
        await sleep(300);
        run();
        return;
      }

      // (2) Observe removal of #page-header
      // (3) If removed, restart from (1)
      observePageHeaderRemoval(pageHeader);
    } catch (err) {
      console.error(err);
    } finally {
      isRunning = false;
    }
  }

  run();
})();
```
