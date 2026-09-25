---
pubDatetime: 2026-04-22T21:28:00+09:00
title: "How to Prevent `\\n` from Appearing in User Input Prompts in Codex"
description: "This article outlines the purpose, usage, and precautions of a UserScript designed to prevent unintended `\n` characters from appearing in the prompt input field when using Codex. It also covers the background, steps, verification methods, and optimized implementation."
---

When using Codex, unwanted `\n` characters may appear in the prompt input field, disrupting the text and making it difficult to input. This section provides a concise overview of the purpose and usage of a UserScript designed to prevent this issue.

## What This Does

This UserScript operates only when pasting into the `#prompt-textarea` element within ChatGPT Codex Cloud.

After pasting, it checks the text nodes of the `<p>` element directly below the input field and normalizes the newline characters as follows:

*   `CRLF` (`\r\n`) → `LF` (`\n`)
*   `CR` (`\r`) → `LF` (`\n`)
*   `LF` (`\n`) → Remains unchanged

In essence, the script preserves the line breaks in the text while ensuring that the newline characters are consistently represented as `LF`.

## Why This Is Useful

The newline characters in text can vary depending on the source operating system or application.

This script aims to standardize the newline characters to `LF` after pasting, ensuring a consistent newline format for text input into Codex.

For example, if the text contains the following string:

```text
AAA\r\nBBB\rCCC\nDDD
```

After normalization, it becomes:

```text
AAA\nBBB\nCCC\nDDD
```

The visual multi-line structure remains intact.

## Optimized Implementation

The previous implementation used `MutationObserver` to monitor the entire page's DOM changes to detect the appearance or replacement of `#prompt-textarea`.

The current implementation eliminates this constant monitoring.

Instead, it registers a single `paste` event listener on the `document` and only processes events originating from `#prompt-textarea`.

This approach offers the following advantages:

*   Avoids constant monitoring of the entire page with `MutationObserver`.
*   Eliminates the need to repeatedly search for the editor using `querySelector`.
*   No need to reattach the listener if `#prompt-textarea` is replaced.
*   Normalization is only performed when a paste event occurs.
*   Avoids rewriting the entire `innerHTML` and only modifies the necessary text nodes.
*   Reduces console log output for debugging.

## How to Use

### Register as a UserScript

Install a browser extension that supports running UserScripts and register the following code as a new UserScript.

Once registered, the script will be automatically enabled when you open the target Codex Cloud page.

### Target Page

This script targets the following URLs:

*   `https://chatgpt.com/codex/cloud`
*   `https://chatgpt.com/codex/cloud/*`
*   `https://chatgpt.com/codex/cloud?*`

## Workflow

The process flow is as follows:

1.  When the page loads, a single `paste` listener is registered on the `document`.
2.  A paste event occurs.
3.  `event.composedPath()` is used to check if the paste event originated within `#prompt-textarea`.
4.  If the event did not originate from the target, no action is taken.
5.  If the event originated from the target, the script waits for the normal paste process to reflect the changes in the DOM.
6.  The `<p>` element directly below `#prompt-textarea` is examined.
7.  For each text node within the `<p>` elements, `CRLF` and `CR` are converted to `LF`.

Since `LF` remains unchanged, the script does not remove existing line breaks.

## UserScript

```javascript
// ==UserScript==
// @name         ChatGPT Codex Cloud - Normalize Newlines to LF
// @namespace    https://chatgpt.com/
// @version      1.2.0
// @description  Normalize CRLF/CR to LF in direct <p> children after paste.
// @match        https://chatgpt.com/codex/cloud
// @match        https://chatgpt.com/codex/cloud/*
// @match        https://chatgpt.com/codex/cloud?*
// @grant        none
// ==/UserScript==

(() => {
  "use strict";

  /**
   * Converts CRLF / CR to LF.
   *
   * \r\n -> \n
   * \r   -> \n
   * \n   -> \n
   */
  function normalizeNewlines(text) {
    return text.replace(/\r\n?/g, "\n");
  }

  /**
   * Processes only the text nodes contained in the <p> element directly below #prompt-textarea.
   */
  function normalizeParagraphs(editor) {
    if (!editor?.isConnected) return;

    for (const p of editor.children) {
      if (p.tagName !== "P") continue;

      const walker = document.createTreeWalker(
        p,
        NodeFilter.SHOW_TEXT
      );

      let node;

      while ((node = walker.nextNode())) {
        const before = node.data;
        const after = normalizeNewlines(before);

        if (before !== after) {
          node.data = after;
        }
      }
    }
  }

  /**
   * Checks if the paste event originated within #prompt-textarea.
   *
   * Uses event delegation, so the listener does not need to be re-registered
   * if the editor is replaced.
   */
  function getEditorFromPasteEvent(event) {
    const path = event.composedPath();

    for (const node of path) {
      if (
        node instanceof Element &&
        node.id === "prompt-textarea"
      ) {
        return node;
      }
    }

    return null;
  }

  /**
   * Registers a single paste listener on the document.
   * Does not use MutationObserver.
   */
  document.addEventListener(
    "paste",
    event => {
      const editor = getEditorFromPasteEvent(event);

      if (!editor) return;

      // Executes after the normal paste process has reflected the changes in the DOM.
      setTimeout(() => {
        normalizeParagraphs(editor);
      }, 0);
    },
    true
  );
})();
```

## Key Points of Newline Normalization

The following code is used for normalization:

```javascript
text.replace(/\r\n?/g, "\n");
```

Here, `\r\n?` first treats `CRLF` (`\r\n`) as a single newline character and also handles a single `CR` (`\r`).

The replacement target is always `\n`.

Therefore, `CRLF` is not incorrectly converted to two `LF` characters, and all newline characters are uniformly converted to `LF`.

## Reasons for Not Rewriting the Entire DOM

This script does not use the method of obtaining and reassigning the `innerHTML` of the paragraph.

Instead, it uses `TreeWalker` to obtain only the text nodes and modifies only the nodes where newline character conversion is actually necessary using `node.data`.

This avoids unnecessarily reconstructing the DOM structure within the paragraph.

## Reasons for Not Using MutationObserver

This processing is only required during pasting.

Therefore, there is no need to constantly monitor the entire page for the appearance, deletion, or replacement of the input field.

By using the `paste` event on the `document` with event delegation, the paste event can be handled even if `#prompt-textarea` is created later or replaced with another DOM node.

## Verification

To verify, paste text with different newline characters into the Codex Cloud input field.

The expected conversion is as follows:

| Before Paste | After Paste |
| --- | --- |
| `A\r\nB` | `A\nB` |
| `A\rB` | `A\nB` |
| `A\nB` | `A\nB` |
| `A\r\nB\rC\nD` | `A\nB\nC\nD` |

Importantly, the script does not remove line breaks or convert the text into a single line; it only unifies the newline characters to `LF`.

## Precautions

### Only Operates During Pasting

This UserScript is triggered by the `paste` event.

It does not constantly monitor and normalize text entered through other methods, such as keyboard input.

### Targets Only the Directly Below `<p>` Element

The processing target is the `<p>` element directly below `#prompt-textarea`.

If the DOM structure of Codex Cloud changes in the future, the target element condition may need to be adjusted.

### Does Not Delete LF

The purpose of this script is not to convert the text into a single line.

Multi-line prompts remain multi-line, and `LF` is retained.

## Summary

This UserScript normalizes `CRLF` and `CR` to `LF` when text is pasted into ChatGPT Codex Cloud.

Since line breaks are preserved, multi-line prompts can still be used.

Furthermore, by using a `paste` event listener on the `document` instead of constantly monitoring the entire page with `MutationObserver`, the script avoids constant monitoring.

The processing also only targets the necessary text nodes, rather than rewriting the entire `innerHTML`, ensuring that the script only performs the required actions at the required time.
