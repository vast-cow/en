---
pubDatetime: 2026-09-25T14:45:00+09:00
title: "How to Hide Annoying Suggestions in the New ChatGPT Chat Interface"
description: "Here's how to use Tampermonkey to hide the unwanted suggestions that appear at the bottom of the new ChatGPT chat screen. This is a simple method that only applies CSS, so it doesn't require any continuous monitoring and runs efficiently."
---

In the new ChatGPT chat interface, suggestions and recommended items may appear at the bottom of the screen.

While sometimes useful, if you don't use them regularly, they can take up space and make the chat interface feel cluttered.

Therefore, I will use Tampermonkey to automatically hide this suggestion section.

## Script to Use

```javascript
// ==UserScript==
// @name         Hide Thread Bottom UL
// @namespace    tampermonkey
// @version      1.1
// @description  Hide the target ul using CSS
// @match        *://*/*
// @run-at       document-start
// @grant        none
// ==/UserScript==

(function () {
    'use strict';

    const style = document.createElement('style');
    style.textContent = `
    #thread-bottom div.contents section ul {
      display: none !important;
    }
  `;

    document.documentElement.appendChild(style);
})();
```

## What This Script Does

This script hides the suggestion list at the bottom of the ChatGPT screen.

The target is the following part:

```css
#thread-bottom div.contents section ul
```

The following is applied to this:

```css
display: none !important;
```

This makes it invisible on the screen.

It doesn't delete the element itself, but rather hides it.

## Registering in Tampermonkey

First, install Tampermonkey on your browser.

Open the Tampermonkey management screen and create a new user script.

Delete the content that is initially entered and paste the above script, then save it.

After that, when you reopen ChatGPT, the target suggestion will be automatically hidden.

## Making It Run Only on ChatGPT

In the above script,

```javascript
// @match        *://*/*
```

Therefore, the script will run on all websites.

If you only use it on ChatGPT, it is easier to use if you change it to the following:

```javascript
// @match        https://chatgpt.com/*
```

The completed version is as follows:

```javascript
// ==UserScript==
// @name         Hide ChatGPT Thread Bottom Suggestions
// @namespace    tampermonkey
// @version      1.1
// @description  Hide the suggestions displayed at the bottom of the ChatGPT screen
// @match        https://chatgpt.com/*
// @run-at       document-start
// @grant        none
// ==/UserScript==

(function () {
    'use strict';

    const style = document.createElement('style');

    style.textContent = `
    #thread-bottom div.contents section ul {
      display: none !important;
    }
  `;

    document.documentElement.appendChild(style);
})();
```

## Why Not Use MutationObserver

In web apps like ChatGPT, the content on the screen may be added dynamically later.

Therefore, it is also possible to consider using MutationObserver to monitor changes on the screen.

However, the purpose of this time is to always hide a specific element.

If you add CSS at the beginning, the same CSS will be automatically applied even if the target element is displayed later.

Therefore, there is no need to constantly monitor changes on the screen.

The process is simple, and you can avoid running unnecessary monitoring processes.

## How to Revert

If you want to revert to the original display, disable or delete this user script in Tampermonkey.

Reload the page, and ChatGPT will return to its original display.

## Notes

This method uses the ChatGPT screen structure.

If the HTML structure or element names are changed by ChatGPT's update,

```css
#thread-bottom div.contents section ul
```

will no longer match, and it will no longer be hidden.

In that case, you need to modify the target CSS selector to match the current screen structure.
