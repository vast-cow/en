---
title: "Switching from the ChatGPT Web Page to the ChatGPT App on iOS"
description: "On iOS, it is possible to move directly from the ChatGPT website in a browser to the ChatGPT app by..."
pubDatetime: 2026-01-19T10:53:35.035Z
---

On iOS, it is possible to move directly from the ChatGPT website in a browser to the ChatGPT app by using a custom URL scheme. The basic idea is to take the current web page address and rewrite it into a format the ChatGPT app can understand, then instruct the device to open that link.

The following JavaScript snippet performs that redirection automatically:

```javascript
javascript:!function(){location.href="chatgpt://"+location.host+location.pathname+location.search+location.hash}();
```

When executed on a ChatGPT web page, the script builds a new URL that starts with `chatgpt://` instead of `https://`. It then appends the current page’s host, path, query parameters, and hash fragment. This preserves the specific page location and any parameters in the URL, so the app can attempt to open the corresponding destination.

In practice, this approach is commonly used as a quick “handoff” method—especially when you are viewing ChatGPT in Safari (or another iOS browser) and want to continue in the native app experience without manually copying and pasting links.
