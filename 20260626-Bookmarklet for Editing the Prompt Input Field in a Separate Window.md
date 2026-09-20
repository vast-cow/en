---
title: "Bookmarklet for Editing the Prompt Input Field in a Separate Window"
description: "This bookmarklet lets you open the prompt input field on a page in a separate window, where you can..."
pubDatetime: 2026-06-26T05:30:45.453Z
---

This bookmarklet lets you open the prompt input field on a page in a separate window, where you can edit it in a larger text area. It reads the current contents of the input field and displays them as the initial value in the editing window. After making your changes, click the `Set` button to apply them back to the original page.

## Purpose

Editing long prompts in a small browser input field can be inconvenient. This bookmarklet opens a dedicated editing window with a larger, easier-to-read text area, making it more comfortable to review and edit your prompt.

Typical use cases include:

* Editing an existing prompt in a separate window
* Making long prompts easier to read and organize
* Working more comfortably with multi-line text
* Applying the edited content back to the original input field

## How to Use

First, save the JavaScript code as a bookmarklet in your browser. Once it has been added, open the target page and run the bookmarklet.

When executed, an editing window opens with the current page title displayed as its heading. The contents of the original page's prompt input field are preloaded into the text area.

After editing the text, click the `Set` button. The updated content will be written back to the prompt input field on the original page, and the editing window will close.

## Automatic Theme Switching

The editing window automatically switches between light and dark themes based on your browser or operating system's theme settings.

The light theme uses a white background with black text, while the dark theme uses a dark background with light-colored text. This allows the editor to match the appearance of your usual environment.

## Loading the Current Value

When the bookmarklet is executed, it first reads the current contents of `#prompt-textarea` on the original page. That content is embedded directly into the generated HTML as the initial value of the `<textarea>`.

As a result, the editing window displays the current prompt immediately when it opens, without needing to retrieve the content from the original page afterward.

## Applying Edited Content

When you click the `Set` button, the text entered in the editing window is split into individual lines and written back to the prompt input field on the original page.

Blank lines are preserved, making it easy to work with multi-line prompts and structured text.

## Cleanup

The editing window is created from a temporary HTML document stored as a `Blob` and opened through its generated URL. Once processing is complete, the temporary URL is released using `URL.revokeObjectURL()`.

This prevents unnecessary temporary URLs from remaining in memory. The URL is also revoked if the popup cannot be opened or if an error occurs during processing.

## Notes

This bookmarklet assumes that the target page contains an element with the ID `#prompt-textarea`. If the element cannot be found, an error message is displayed and no further processing is performed.

If your browser blocks pop-up windows, the editing window may fail to open. In that case, you will need to allow pop-ups for the target page.

```javascript
(() => {
  const escapeHtml = (s) =>
    (s || '')
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;');
  const promptTextarea = document.querySelector('#prompt-textarea');
  if (!promptTextarea) {
    alert('Could not find #prompt-textarea');
    return;
  }
  const escapedParentTitle = escapeHtml(document.title || 'Prompt Setter');
  const initialText = ((el) => {
    const nodes = Array.from(el.childNodes);
    return nodes.length
      ? nodes
          .map((node) =>
            3 === node.nodeType
              ? node.textContent || ''
              : 'BR' === node.nodeName
                ? ''
                : node.textContent || '',
          )
          .join('\n')
      : el.innerText || el.textContent || '';
  })(promptTextarea);
  const blob = new Blob(
    [
      `<!doctype html>\n<html lang="en">\n<head>\n  <meta charset="utf-8">\n  <title>${escapedParentTitle}</title>\n  <style>\n    :root {\n      color-scheme: light dark;\n      --bg: #ffffff;\n      --fg: #111111;\n      --muted: #333333;\n      --border: #cccccc;\n      --textarea-bg: #ffffff;\n      --textarea-fg: #111111;\n      --button-bg: #f3f3f3;\n      --button-fg: #111111;\n      --button-border: #999999;\n    }\n\n    @media (prefers-color-scheme: dark) {\n      :root {\n        --bg: #111111;\n        --fg: #eeeeee;\n        --muted: #cccccc;\n        --border: #444444;\n        --textarea-bg: #1e1e1e;\n        --textarea-fg: #eeeeee;\n        --button-bg: #2a2a2a;\n        --button-fg: #eeeeee;\n        --button-border: #666666;\n      }\n    }\n\n    body {\n      font-family: sans-serif;\n      margin: 16px;\n      background: var(--bg);\n      color: var(--fg);\n    }\n\n    .page-title {\n      font-size: 24px;\n      font-weight: 700;\n      line-height: 1.4;\n      margin: 0 0 16px;\n      word-break: break-word;\n    }\n\n    textarea {\n      width: 100%;\n      height: 240px;\n      box-sizing: border-box;\n      font-family: monospace;\n      font-size: 14px;\n      background: var(--textarea-bg);\n      color: var(--textarea-fg);\n      border: 1px solid var(--border);\n    }\n\n    button {\n      margin-top: 12px;\n      padding: 8px 16px;\n      font-size: 14px;\n      background: var(--button-bg);\n      color: var(--button-fg);\n      border: 1px solid var(--button-border);\n      border-radius: 4px;\n      cursor: pointer;\n    }\n\n    button:hover {\n      filter: brightness(0.96);\n    }\n\n    @media (prefers-color-scheme: dark) {\n      button:hover {\n        filter: brightness(1.15);\n      }\n    }\n\n    .msg {\n      margin-top: 12px;\n      color: var(--muted);\n      white-space: pre-wrap;\n    }\n  </style>\n</head>\n<body>\n  <div class="page-title">${escapedParentTitle}</div>\n  <textarea id="usertext" placeholder="Enter text here">${escapeHtml(initialText)}</textarea>\n  <br>\n  <button id="setButton" type="button">Set</button>\n  <div class="msg" id="msg"></div>\n</body>\n</html>`,
    ],
    { type: 'text/html' },
  );
  const url = URL.createObjectURL(blob);
  let revoked = !1;
  const revokeUrl = () => {
    if (!revoked) {
      URL.revokeObjectURL(url);
      revoked = !0;
    }
  };
  const child = window.open(url, '_blank');
  if (!child) {
    revokeUrl();
    alert('Could not open popup');
    return;
  }
  const setup = () => {
    try {
      const childDoc = child.document;
      const usertextEl = childDoc.getElementById('usertext');
      const setButton = childDoc.getElementById('setButton');
      const msgEl = childDoc.getElementById('msg');
      if (!usertextEl || !setButton || !msgEl) {
        setTimeout(setup, 50);
        return;
      }
      try {
        child.addEventListener('beforeunload', revokeUrl, { once: !0 });
      } catch (_) {}
      setButton.addEventListener('click', async () => {
        try {
          if (!child.opener || child.opener.closed)
            throw new Error('Cannot access parent window');
          const openerDoc = child.opener.document;
          const promptTextarea = openerDoc.querySelector('#prompt-textarea');
          if (!promptTextarea)
            throw new Error('Could not find #prompt-textarea');
          const usertext = usertextEl.value;
          const newNodes = usertext
            .replace(/\r\n/g, '\n')
            .split('\n')
            .map((promptLine) => {
              const p = openerDoc.createElement('p');
              '' === promptLine
                ? p.appendChild(openerDoc.createElement('br'))
                : (p.textContent = promptLine);
              return p;
            });
          promptTextarea.replaceChildren(...newNodes);
          await new Promise((resolve) => setTimeout(resolve, 0));
          promptTextarea.lastChild &&
            promptTextarea.lastChild.scrollIntoView &&
            promptTextarea.lastChild.scrollIntoView();
          revokeUrl();
          child.close();
        } catch (err) {
          msgEl.textContent = 'Error: ' + err.message;
          revokeUrl();
        }
      });
    } catch (e) {
      setTimeout(setup, 50);
    }
  };
  setup();
})();
```
