---
title: "How to reliably open YouTube links in the YouTube app using an iOS Shortcut"
description: "When you open a YouTube URL copied from Safari or other apps as-is, it may play in a browser or..."
pubDatetime: 2026-01-19T13:39:16.935Z
---

When you open a YouTube URL copied from Safari or other apps as-is, it may play in a browser or display in an unintended format.
This shortcut **parses the YouTube URL in your clipboard and converts it into a YouTube-app URL scheme (`vnd.youtube://`) before opening it**.

It supports these three cases:

* `youtube.com/shorts/...` (Shorts)
* `youtube.com/watch?v=...` (Standard video)
* Anything else (fallback: replace only the scheme and try opening in the app)



## Overall flow

1. Get the URL string from the clipboard
2. Use a regular expression to check if it’s a Shorts URL → extract the video ID
3. Use a regular expression to check if it’s a standard watch URL → extract the video ID
4. Build `vnd.youtube://...` based on which ID was obtained
5. Open the final URL



## Steps (Action setup)

### 1) Get the clipboard content

* Action: **Text**
* Content: **Clipboard**
* In this guide, the resulting string is called **`inputUrl`**

> From here on, the shortcut determines what kind of YouTube URL `inputUrl` is, then converts it into an appropriate “open in YouTube app” URL.



### 2) Detect whether it’s a Shorts URL and extract the video ID

* Action: **Match Text** (against `inputUrl`)
* Regular expression:

`^https?://(?:www\.|m\.)?youtube\.com/shorts/([\w-]{11})(?:\?.*)?`

* If it matches, retrieve the capture group (video ID):

  * Action: **Get Group from Matched Text**
  * Group index: **1**
  * Call the extracted 11-character ID **`shortsId`**

If `shortsId` is obtained, it is confirmed as Shorts.



### 3) Detect whether it’s a standard video (watch?v=) URL and extract the video ID

In case it’s not Shorts, also check the standard video URL format.

* Action: **Match Text** (against `inputUrl`)
* Regular expression:

`^https?://(?:www\.|m\.)?youtube\.com/watch\?[^#]*\bv=([A-Za-z0-9_-]{11})`

* If it matches, retrieve the capture group (video ID):

  * Action: **Get Group from Matched Text**
  * Group index: **1**
  * Call the extracted ID **`videoId`**



## 4) Use conditional branching to build the URL to open

This is the key part of the shortcut: create a YouTube-app URL depending on which ID you have.

### 4-1) If a Shorts ID was obtained

* Condition: **`shortsId` has any value**
* Action: **URL**
* Content:

`vnd.youtube://youtube.com/shorts/{shortsId}?`

Treat this as **`openUrl`**.



### 4-2) If it’s not Shorts and a standard video ID was obtained

* Condition: **`videoId` has any value**
* Action: **URL**
* Content:

`vnd.youtube://youtu.be/{videoId}`

This is also **`openUrl`**.



### 4-3) If neither ID was obtained (fallback)

The URL might be in an unexpected format (non-standard share URL, different domain, unusual parameters, etc.).
In this case, **replace only the URL scheme (`https://`, etc.) with `vnd.youtube://`** and try launching the app.

* Action: **Replace Text**
* Input: `inputUrl`
* Replacement (Regex enabled):

  * Find: `^[^:]+\://`
  * Replace with: `vnd.youtube://`

Call the result **`fallbackUrlText`**.

Then:

* Action: **URL**
* Content: `fallbackUrlText`

This becomes the fallback **`openUrl`**.



### 5) Open the final URL

Finally, open the URL created above (`openUrl`).

* Action: **Open URLs**
* Target: `openUrl`



## Summary: What this shortcut enables

* Open YouTube Shorts **directly in the YouTube app**
* For standard videos (`watch?v=`), **extract the ID and reliably open in the app**
* Even for unexpected URL formats, **attempt to launch the app by swapping the scheme**
