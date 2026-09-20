---
title: "Copy Multiple Files to Markdown with One Script for GitHub Gist"
description: "This JavaScript snippet is a self-invoking utility designed to collect multiple file entries from a..."
pubDatetime: 2026-01-19T08:54:22.358Z
updatedDate: 2026-01-19T14:36:26.607Z
---

This JavaScript snippet is a self-invoking utility designed to collect multiple file entries from a web page and copy them to the clipboard as a clean, Markdown-formatted document. It is especially useful when working with interfaces that display several editable files, such as code editors or snippet managers.

* It scans the page for elements marked as files.
* For each file, it extracts the displayed file name and its text content.
* It wraps each file’s content in Markdown-style code fences.
* It combines all files into a single Markdown string, separated by blank lines.
* It copies the final result directly to the clipboard using a temporary textarea.

The script runs entirely in the browser, requires no external libraries, and provides a simple console message indicating whether the copy operation succeeded. This makes it a practical, lightweight solution for quickly exporting structured content for documentation or sharing.

```javascript
javascript:(()=>{const md=Array.from(document.querySelectorAll(%22.file%22)).map(file=>{const nameEl=file.querySelector(%22:scope > .file-header > .file-info > a > strong%22);const fileName=(nameEl?.innerText||%22%22).trim();const%20textEl=file.querySelector(%27:scope%20%3E%20textarea[name=%22gist[content]%22]%27);const%20content=(textEl?.value??%22%22).trimEnd();const%20delimiter=content.includes(%22`%22.repeat(3))?%22~%22.repeat(3):%22`%22.repeat(3);let%20lang=%22%22;fileName.endsWith(%22.md%22)?lang=%22markdown%22:fileName.endsWith(%22.py%22)?lang=%22python%22:fileName.endsWith(%22.js%22)?lang=%22javascript%22:fileName.endsWith(%22.service%22)||fileName.endsWith(%22.timer%22)?lang=%22ini%22:fileName.endsWith(%22.c%22)||fileName.endsWith(%22.h%22)?lang=%22c%22:fileName.endsWith(%22.cpp%22)||fileName.endsWith(%22.hpp%22)?lang=%22c++%22:fileName.endsWith(%22.json%22)?lang=%22json%22:fileName.endsWith(%22.toml%22)&&(lang=%22toml%22);return%22##%20%60%22+fileName+%22%60\n\n%22+delimiter+lang+%22\n%22+content+%22\n%22+delimiter+%22\n%22}).filter(Boolean).join(%22\n\n%22);const%20ta=document.createElement(%22textarea%22);ta.value=md;ta.setAttribute(%22readonly%22,%22%22);ta.style.position=%22fixed%22;ta.style.top=%22-9999px%22;ta.style.left=%22-9999px%22;document.body.appendChild(ta);ta.focus();ta.select();const%20ok=document.execCommand(%22copy%22);document.body.removeChild(ta);console.log(ok?%22Copied%20to%20clipboard.%22:%22Copy%20failed.%22)})();
```

```javascript
(() => {
  const md = Array.from(document.querySelectorAll('.file'))
    .map((file) => {
      const nameEl = file.querySelector(
        ':scope > .file-header > .file-info > a > strong',
      );
      const fileName = (nameEl?.innerText || '').trim();
      const textEl = file.querySelector(
        ':scope > textarea[name="gist[content]"]',
      );
      const content = (textEl?.value ?? '').trimEnd();
      const delimiter = content.includes('`'.repeat(3))
        ? '~'.repeat(3)
        : '`'.repeat(3);
      let lang = '';
      fileName.endsWith('.md')
        ? (lang = 'markdown')
        : fileName.endsWith('.py')
          ? (lang = 'python')
          : fileName.endsWith('.js')
            ? (lang = 'javascript')
            : fileName.endsWith('.service') || fileName.endsWith('.timer')
              ? (lang = 'ini')
              : fileName.endsWith('.c') || fileName.endsWith('.h')
                ? (lang = 'c')
                : fileName.endsWith('.cpp') || fileName.endsWith('.hpp')
                  ? (lang = 'c++')
                  : fileName.endsWith('.json')
                    ? (lang = 'json')
                    : fileName.endsWith('.toml') && (lang = 'toml');
      return (
        '## `' +
        fileName +
        '`\n\n' +
        delimiter +
        lang +
        '\n' +
        content +
        '\n' +
        delimiter +
        '\n'
      );
    })
    .filter(Boolean)
    .join('\n\n');
  const ta = document.createElement('textarea');
  ta.value = md;
  ta.setAttribute('readonly', '');
  ta.style.position = 'fixed';
  ta.style.top = '-9999px';
  ta.style.left = '-9999px';
  document.body.appendChild(ta);
  ta.focus();
  ta.select();
  const ok = document.execCommand('copy');
  document.body.removeChild(ta);
  console.log(ok ? 'Copied to clipboard.' : 'Copy failed.');
})();
```
