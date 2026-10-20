---
title: "How to Enable Dark Mode for Specific Websites Using Tampermonkey (Automatically Linked to OS Settings)"
description: "A Tampermonkey user script that applies a CSS filter to invert page colors in dark mode for chosen URLs, without complex JavaScript."
pubDatetime: 2026-10-20T12:36:00+09:00
---

Have you ever wanted to automatically enable dark mode for websites that don't natively support it, based on your browser's settings?

While more and more websites are offering dark mode, many still only provide a light mode option.

This article introduces a user script that uses the browser extension **Tampermonkey** to apply dark mode to specific websites.

This method has the following features:

- Automatically linked to the OS and browser's dark mode settings.
- Inverts the colors of web pages only in dark mode.
- Allows you to specify multiple URLs to limit the target websites.
- Doesn't require complex JavaScript processing.
- Automatically disables color inversion when switching back to light mode.

## 1. How it Works

This method uses the CSS `filter` property to invert the colors of the entire web page.

The CSS used is as follows:

```css
html {
    filter: invert(1) hue-rotate(180deg) !important;
}
```

`invert(1)` is a CSS filter that inverts the colors. A white background becomes black, and black text becomes white.

However, simply inverting the colors can significantly alter the hue, such as turning blue into yellow.

To address this, we combine it with `hue-rotate(180deg)`, which rotates the hue by 180 degrees, bringing the colors closer to their original appearance.

Furthermore, we use the CSS `prefers-color-scheme` media query.

```css
@media (prefers-color-scheme: dark) {
    html {
        filter: invert(1) hue-rotate(180deg) !important;
    }
}
```

This ensures that the filter is only applied when the browser is using dark mode.

## 2. Install Tampermonkey

First, install Tampermonkey on your browser.

Tampermonkey is a browser extension that allows you to run custom JavaScript on any website.

Official website:

https://www.tampermonkey.net/

After installing, select "Add a new script" from the Tampermonkey menu.

## 3. User Script

Paste the following code into the new script's editor and save it.

```javascript
// ==UserScript==
// @name         Auto Dark Mode (Invert)
// @namespace    https://tampermonkey.net/
// @version      1.1
// @description  Apply invert filter on selected sites in dark mode
// @match        https://example.com/*
// @match        https://example.org/*
// @match        https://sub.example.net/*
// @run-at       document-start
// @grant        GM_addStyle
// ==/UserScript==

(function () {
    'use strict';

    GM_addStyle(`
        html {
          background-color: #fff !important;
        }
        @media (prefers-color-scheme: dark) {
            html {
                filter: invert(1) hue-rotate(180deg) !important;
            }
        }
    `);
})();
```

This script uses `GM_addStyle()` to add CSS to the page.

By specifying `@run-at document-start`, the styles are applied from the early stages of page loading, minimizing the color-shifting effect that can occur after the page has loaded.

## 4. Specify the URLs to Apply

In this script, Tampermonkey's `@match` is used to specify the websites to which dark mode will be applied.

For example:

```javascript
// @match        https://example.com/*
// @match        https://example.org/*
// @match        https://sub.example.net/*
```

You can specify multiple `@match` lines to add the target websites.

Here are some examples of how to specify the target:

| Description | Target |
|---|---|
| `https://example.com/*` | All pages of example.com |
| `https://*.example.com/*` | example.com and its subdomains |
| `https://example.com/docs/*` | Pages under the docs directory |
| `*://example.com/*` | Both HTTP and HTTPS |
| `https://example.com/page` | A specific page |

For example, to target GitHub and Stack Overflow, you would write:

```javascript
// @match        https://github.com/*
// @match        https://stackoverflow.com/*
```

It is recommended to specify only the websites that **do not have native dark mode support**, rather than all websites.

## 5. Reason for Specifying the Background Color to White

In this script, in addition to the color inversion settings, the following CSS is also specified:

```css
html {
    background-color: #fff !important;
}
```

This is to explicitly set the background color of the HTML element to white.

When using the color inversion filter, a white background is displayed as black.

However, on pages with transparent backgrounds, the expected background color may not be displayed.

Therefore, by setting the `html` background color to white in advance, it is displayed as a black background in dark mode.

Note that this background color specification is outside the media query, so it is applied even in light mode.

Depending on the site, it may overwrite the original background color, so adjust it as needed.

## 6. Automatic Linking to OS Dark Mode Settings

The advantage of this script is that it automatically links to the OS and browser's color scheme.

For example, when you enable dark mode in Windows or macOS, `prefers-color-scheme: dark` is matched, and the color inversion filter is applied.

Conversely, when you switch back to light mode, the filter is disabled.

There is no need to monitor setting changes with JavaScript's `matchMedia()`.

Because it works with only CSS media queries, the configuration is simple.

## 7. Precautions

While this method provides an easy way to enable dark mode, there are some precautions to keep in mind.

### Images and Videos Also Change Color

Because the filter is applied to the entire `html` element, not only text and backgrounds but also images and videos are affected.

The `hue-rotate(180deg)` partially corrects the color shift, but the colors will not be exactly the same as the original image.

In particular, with photos, the colors and brightness may appear unnatural.

### Not Suitable for Sites That Already Have Dark Mode

If a website provides its own dark mode, the already dark background may be inverted, causing it to appear bright.

Therefore, it is appropriate to specify only websites that do not have native dark mode support with `@match`.

### Display May Be Distorted Depending on the Site's Design

CSS filters affect the overall rendering of the page.

Depending on the site, problems may occur with the display of fixed-position elements, pop-ups, and background images.

Also, separate handling may be required for content within iframes.

### `background-color` is Always Applied

In this script, the `html` background color is always set to white, even in light mode.

If you want to prioritize the normal display, you can move this specification into the dark mode media query.

## Summary

By combining Tampermonkey and CSS's `prefers-color-scheme`, you can easily enable dark mode for websites that do not natively support it.

This method is a simple way to enable dark mode using CSS's `filter`, and it does not individually adjust the site's original color scheme.

Therefore, there may be problems with the colors of images and some UI elements, but the fact that you don't have to manually set background colors and text colors is a major advantage.

In particular, it is suitable for the purpose of **automatically enabling dark mode for specific websites that you use regularly, based on your OS settings**.

You can achieve this with just a few lines of CSS and Tampermonkey without making major changes to the existing site's CSS, so try it out on websites that don't have native dark mode support.
