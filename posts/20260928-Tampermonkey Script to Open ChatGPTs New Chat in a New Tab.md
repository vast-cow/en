---
pubDatetime: 2026-09-28T01:43:32+09:00
title: "Tampermonkey Script to Open ChatGPT's 'New Chat' in a New Tab"
description: "Allows you to start a new conversation with a single click while preserving the current conversation. This small customization is useful for those who often handle multiple topics simultaneously."
---

When using ChatGPT, I've always wanted a **"Open New Chat in a New Tab" button**.

Normally, clicking "New Chat" simply switches the current tab to the new chat.

However, I often find myself wanting to:

- Keep the current conversation open while starting a new one.
- Open multiple chats side-by-side during research.
- Create a new tab with a single click, without having to consciously use Ctrl/Cmd + Click.

For these use cases, a dedicated button would be very helpful.

Therefore, I created a UserScript using Tampermonkey that **automatically adds a "New Tab" button next to ChatGPT's "New Chat" button**.

## Expected Result

On the ChatGPT screen, a **"New Tab"** button is added next to the **"New Chat"** button.

Clicking "New Tab" keeps the currently open chat intact and opens a new ChatGPT chat in a separate tab.

This is very convenient when having multiple conversations with ChatGPT about different topics simultaneously.

## Tampermonkey Script

Register the following script in Tampermonkey:

```javascript
// ==UserScript==
// @name         Add New Chat Link
// @namespace    tampermonkey
// @version      1.0
// @description  「新しいチャット」ボタンを複製してリンクを追加
// @match        https://chatgpt.com/*
// @grant        none
// ==/UserScript==

(function () {
    'use strict';

    const TARGET_TEXT = '新しいチャット';
    const CLONED_TEXT = '新規タブ';
    const ADDED_LINK_SELECTOR = 'a[data-tm-new-chat-link]';

    /**
     * root 自身または子孫から
     * 「新しいチャット」の span を探す
     */
    function findTargetSpan(targetRoot) {
        // root 自身を確認
        if (
            targetRoot instanceof HTMLSpanElement &&
            targetRoot.innerText.trim() === TARGET_TEXT
        ) {
            return targetRoot;
        }

        // 子孫を確認
        if (targetRoot.querySelectorAll) {
            for (const span of targetRoot.querySelectorAll('span')) {
                if (span.innerText.trim() === TARGET_TEXT) {
                    return span;
                }
            }
        }

        return null;
    }

    function initialize() {
        const root = document.querySelector('#root');

        if (!root) {
            console.warn('[Tampermonkey] #root not found');
            return;
        }

        const observerOptions = {
            childList: true,
            subtree: true
        };

        let observer;

        function process(targetRoot) {
            // process 中は observer を停止
            observer?.disconnect();

            try {
                const span = findTargetSpan(targetRoot);

                if (!span) {
                    return;
                }

                // 祖先の button を取得
                const button = span.closest('button');

                if (!button) {
                    return;
                }

                // button の parent を取得
                const parent = button.parentElement;

                if (!parent) {
                    return;
                }

                // すでに追加済みなら何もしない
                if (parent.querySelector(ADDED_LINK_SELECTOR)) {
                    return;
                }

                // button を複製
                const buttonCloned = button.cloneNode(true);

                // 複製した button 内の
                // 「新しいチャット」を「新規Window」に変更
                const clonedSpan = findTargetSpan(buttonCloned);

                if (clonedSpan) {
                    clonedSpan.innerText = CLONED_TEXT;
                }

                buttonCloned.classList.replace("bg-primary-ghost-hover", "hover:bg-primary-ghost-hover")
                buttonCloned.classList.add("data-[state=open]:bg-primary-ghost-hover")
                buttonCloned.firstChild.classList.replace("text-emphasis", "text-default")

                // <a href="/"> を作成
                const anchor = document.createElement('a');

                anchor.href = '/';
                anchor.target = "_blank";
                anchor.dataset.tmNewChatLink = 'true';

                anchor.appendChild(buttonCloned);

                // 元 button の parent に追加
                parent.prepend(anchor);

                console.log('[Tampermonkey] link added:', anchor);
            } finally {
                // process 完了後に監視を再開
                observer?.observe(root, observerOptions);
            }
        }

        observer = new MutationObserver((mutations) => {
            // callback 開始時点で止める
            observer.disconnect();

            try {
                for (const mutation of mutations) {
                    for (const node of mutation.addedNodes) {
                        if (!(node instanceof Element)) {
                            continue;
                        }

                        process(node);
                    }
                }
            } finally {
                observer.observe(root, observerOptions);
            }
        });

        // 初回実行
        process(root);

        // #root 以下への子孫要素追加を監視
        observer.observe(root, observerOptions);
    }

    if (document.readyState === 'complete') {
        initialize();
    } else {
        window.addEventListener('load', initialize, { once: true });
    }
})();
```

## What it Does

The mechanism itself is simple.

It finds the `span` element containing the text "New Chat" on the ChatGPT screen and retrieves its parent `button`.

```javascript
const span = findTargetSpan(targetRoot);
const button = span.closest('button');
```

Then, it clones the entire button.

```javascript
const buttonCloned = button.cloneNode(true);
```

Instead of writing CSS from scratch, the script **clones the existing ChatGPT button**, which is key.

This results in an appearance that is relatively consistent with ChatGPT's UI.

The cloned button changes the display from:

```text
New Chat
```

to:

```text
New Tab
```

```javascript
const clonedSpan = findTargetSpan(buttonCloned);

if (clonedSpan) {
    clonedSpan.innerText = CLONED_TEXT;
}
```

## Opening ChatGPT in a New Tab

The cloned button is placed inside an `a` element.

```javascript
const anchor = document.createElement('a');

anchor.href = '/';
anchor.target = '_blank';
```

The key part is:

```javascript
anchor.target = '_blank';
```

This ensures that when the button is pressed, instead of replacing the current ChatGPT screen, it **opens ChatGPT in a new tab**.

You can start a new conversation while keeping the current chat open.

## Why MutationObserver is Used

ChatGPT differs slightly from typical websites that reload the entire page with each screen transition; its screen content is dynamically changed by JavaScript.

Therefore, simply using:

```javascript
window.addEventListener('load', ...)
```

to add the button may cause the added button to disappear when ChatGPT redraws the UI.

This is why `MutationObserver` is used.

```javascript
observer = new MutationObserver((mutations) => {
    // ...
});
```

And:

```javascript
observer.observe(root, {
    childList: true,
    subtree: true
});
```

This monitors for DOM elements added under `#root`.

Even if ChatGPT updates the UI, the script will find "New Chat" and, if necessary, add "New Tab" again.

## Preventing Excessive Button Creation

When using `MutationObserver`, it is important to be aware that it also monitors the results of your own DOM changes.

Therefore, this script attaches the following marker to the added link:

```javascript
anchor.dataset.tmNewChatLink = 'true';
```

In HTML, this looks like:

```html
<a data-tm-new-chat-link="true">
```

And:

```javascript
if (parent.querySelector(ADDED_LINK_SELECTOR)) {
    return;
}
```

This prevents adding the button if it already exists.

Furthermore, the `process()` function temporarily stops the `MutationObserver` and then resumes monitoring after the process is complete.

```javascript
observer?.disconnect();

try {
    // DOM manipulation
} finally {
    observer?.observe(root, observerOptions);
}
```

This prevents the script from picking up its own DOM changes and repeatedly processing them.

## Installation Method

If you already have Tampermonkey installed, the setup is easy.

1. Open the Tampermonkey dashboard.
2. Select "Add a new script."
3. Delete the code that is initially present.
4. Paste the UserScript above.
5. Save.
6. Reload ChatGPT.

This will add "New Tab" to the ChatGPT screen.

## Recommended for These Users

This UserScript is particularly useful for those who use ChatGPT to **manage multiple chats simultaneously, rather than asking all questions in a single chat.**

For example, you can use it like this:

```text
Tab 1: Ask questions about programming
Tab 2: Create email drafts
Tab 3: Research technical information
Tab 4: Brainstorming
```

It makes it easier to use ChatGPT in this way, as you can create the next chat without leaving the current chat.

The more you use ChatGPT, the more benefits you will get.

## Notes

This UserScript relies on the ChatGPT DOM structure.

Specifically, it relies on ChatGPT's HTML structure, such as:

```javascript
#root
```

and:

```javascript
span.closest('button')
```

Therefore, it may stop working if ChatGPT's UI is updated.

Also, since it searches for the Japanese text "New Chat," it will not work as is if you are using ChatGPT in another language, such as English.

For example, to support the English UI, you would need to modify the script to also support searching for "New chat."

Tampermonkey UserScripts are a mechanism for executing JavaScript on web pages, so it is important to only install scripts that you understand.

## Summary

By installing this UserScript, you can add a **"New Tab" button** next to the "New Chat" button on ChatGPT.

While the changes are small, they reduce the action of:

**"Keep the current chat open → Open a new tab → Open ChatGPT"**

to:

**"Click the 'New Tab' button once."**

This Tampermonkey script is especially useful for those who often handle multiple topics simultaneously in ChatGPT. If you use ChatGPT frequently in a browser, it's worth trying.
