---
title: "Showing the Model Name in ChatGPT with User-Agent Switcher and Manager"
description: "This article explains a simple way to make the model name appear more explicitly in the ChatGPT..."
pubDatetime: 2026-04-13T10:03:22.107Z
---

This article explains a simple way to make the model name appear more explicitly in the ChatGPT interface by using the **User-Agent Switcher and Manager** Chrome extension with a custom setting for `chatgpt.com`.

## Overview

By configuring **User-Agent Switcher and Manager** to send a mobile Chrome-on-iPhone user agent only for `chatgpt.com`, the ChatGPT site can display the model more clearly in the model selection area.

The key idea is to apply a custom user agent to a specific domain rather than changing the browser identity globally.

## Configuration Used

The following JSON configuration was used in the extension:

```json
{
  "blacklist": [],
  "custom": {
    "chatgpt.com": "Mozilla/5.0 (iPhone; CPU iPhone OS 13_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) CriOS/146.0.0.0 Mobile/15E148 Safari/604.1"
  },
  "last-update": 1776073925509,
  "mode": "custom",
  "parser": {},
  "popular-browsers": [
    "Internet Explorer",
    "Safari",
    "Chrome",
    "Firefox",
    "Opera",
    "Edge",
    "Vivaldi"
  ],
  "popular-oss": [
    "Windows",
    "Mac OS",
    "Linux",
    "Chromium OS",
    "Android"
  ],
  "protected": [
    "google.com/recaptcha",
    "gstatic.com/recaptcha",
    "accounts.google.com",
    "accounts.youtube.com",
    "gitlab.com/users/sign_in",
    "challenges.cloudflare.com"
  ],
  "remote-address": "",
  "user-styling": "",
  "userAgentData": true,
  "whitelist": [],
  "json-guid": "8455108d-186e-47c2-89ec-41d7384aec1e",
  "json-forced": false
}
```

## What This Setting Does

The important part of the configuration is this entry:

```json
"custom": {
  "chatgpt.com": "Mozilla/5.0 (iPhone; CPU iPhone OS 13_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) CriOS/146.0.0.0 Mobile/15E148 Safari/604.1"
}
```

This tells the extension to use a specific **iPhone Chrome user agent** when visiting `chatgpt.com`. Because the setting is domain-specific, it does not affect unrelated websites.

The extension is also set to:

```json
"mode": "custom"
```

That means the browser applies the custom rules defined in the `custom` section.

## Why the Model Name Becomes Visible

With this setup, ChatGPT may serve a different interface variant based on the browser identity it detects. In this case, the model selection area can show the model name more explicitly.

In practical terms, changing the user agent causes the site to treat the browser more like a mobile client, and that can alter how the interface presents model information.

## Important Details

### Site-Specific Behavior

This configuration only targets:

* `chatgpt.com`

It does not apply the iPhone user agent to every site.

### Protected Domains

The `protected` list includes domains such as reCAPTCHA, Google sign-in pages, and Cloudflare challenge pages. These are excluded from spoofing to avoid login or verification problems.

### User-Agent Client Hints

The setting:

```json
"userAgentData": true
```

indicates that user-agent-related client hint behavior is also enabled. This can matter because modern websites may use more than the classic user agent string when deciding which interface to show.

## Practical Result

After applying this configuration in **User-Agent Switcher and Manager**, ChatGPT can be displayed in a way that makes the model shown in the model selector easier to identify.

This is a lightweight workaround for users who want clearer visibility of the active model without changing their full browsing environment.

## Conclusion

Using **User-Agent Switcher and Manager** with a custom entry for `chatgpt.com` is a simple method for changing how ChatGPT presents its interface. With the iPhone Chrome user agent shown above, the model name can appear explicitly in the model selection area, making it easier to confirm which model is currently being used.
