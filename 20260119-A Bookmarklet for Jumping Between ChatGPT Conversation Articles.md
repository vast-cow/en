---
title: "A Bookmarklet for Jumping Between ChatGPT Conversation Articles"
description: "Overview   The content describes a JavaScript bookmarklet designed to help users jump..."
pubDatetime: 2026-01-19T08:40:30.752Z
---

## Overview

The content describes a JavaScript **bookmarklet** designed to help users jump quickly to specific parts of a ChatGPT conversation. On pages where the conversation is rendered as multiple `<article>` elements, the bookmarklet detects those articles, presents them in a searchable interface, and scrolls smoothly to the selected target.

In practical terms, it adds a lightweight “navigation panel” on top of the current page so you can filter and select an article instead of manually scrolling.

## What the Bookmarklet Does

### Scans the Page for Conversation Segments

After execution, the script:

* Searches the current document for all `<article>` elements.
* If none are found, it shows an alert indicating there is nothing to navigate.
* Builds a list of “jump targets,” where each target includes:

  * Its index (1, 2, 3, …)
  * The `<article>` DOM element reference
  * A label generated from the first portion of the article’s text content

To keep labels readable, the script normalizes whitespace and truncates the preview to a fixed length.

### Displays a Modal Overlay Navigator

The bookmarklet creates a full-screen overlay with a centered modal panel. This UI includes:

* A header displaying a title like `Article Jump (N)` where `N` is the number of detected articles
* A search input (implemented as an ARIA combobox)
* A large scrollable listbox showing the matching articles
* A “Go” button for explicit navigation
* Footer hints showing keyboard shortcuts

If a previous instance of the UI already exists, it is removed first so the bookmarklet can be run repeatedly without stacking multiple overlays.

## User Interaction and Accessibility Features

### Search and Filtering

As the user types in the search box, the list filters in real time by substring matching against each label. If there are no matches, the list shows a “No matches” message.

### Keyboard Navigation

The UI is designed to be operable without a mouse:

* **Up / Down arrows** move the active selection within the list
* **Enter** jumps to the currently selected article
* **Escape** closes the overlay

The script also maintains accessibility attributes such as:

* `role="dialog"` and `aria-modal="true"` for the modal container
* `role="combobox"` and related ARIA attributes on the input
* `role="listbox"` and `role="option"` for the results list
* `aria-selected` for selection state
* `aria-activedescendant` to reflect the active option while navigating

### Mouse Interaction

Users can also:

* Hover over items to update the active highlight
* Click an item to select and jump
* Click outside the panel (on the dimmed overlay) to close the UI
* Use the “Close” button in the header

## How the Jump Works

When an article is selected, the script:

1. Closes the overlay cleanly (removing event listeners and DOM nodes).
2. Calls `scrollIntoView` with smooth scrolling to bring the chosen `<article>` to the top of the viewport.
3. Temporarily adds a visible outline to the target article to make it easy to spot.
4. Removes the outline after a short delay.

This combination provides both navigation and a brief visual confirmation of where the user landed.

## Error Handling and Cleanup

The bookmarklet is wrapped in a `try/catch` block. If any unexpected error occurs during execution, it shows an alert with a descriptive message.

It also implements a dedicated cleanup routine that:

* Removes the overlay from the DOM
* Detaches event listeners from the overlay and document

This prevents the UI from lingering or leaving behind handlers that could interfere with normal page behavior.

## Summary

This bookmarklet provides a practical, accessible way to navigate long ChatGPT conversations by treating each `<article>` element as a jump destination. With a searchable list, keyboard controls, and smooth scrolling plus highlighting, it effectively turns a lengthy scroll experience into a fast “type-and-jump” workflow.

```javascript
(() => {
  try {
    const KEY = '__article_jump_ui__';
    const old = document.getElementById(KEY);
    if (old) old.remove();

    const arts = Array.from(document.querySelectorAll('article'));
    if (!arts.length) {
      alert('No <article> elements found');
      return;
    }

    const normalize = (s) =>
      String(s ?? '')
        .replace(/\s+/g, ' ')
        .trim();

    const head = (t) => {
      const s = normalize(t);
      return s.length ? s.slice(0, 180) : '(no text)';
    };

    const items = arts.map((a, i) => ({
      i,
      el: a,
      label: `${i + 1}. ${head(a.textContent)}`,
    }));

    // Root overlay
    const overlay = document.createElement('div');
    overlay.id = KEY;
    overlay.style.cssText =
      'position:fixed;inset:0;z-index:2147483647;background:rgba(0,0,0,.55);display:flex;align-items:center;justify-content:center;padding:18px;';

    // Modal panel (wider)
    const panel = document.createElement('div');
    panel.setAttribute('role', 'dialog');
    panel.setAttribute('aria-modal', 'true');
    panel.setAttribute('aria-label', 'Article Jump');
    panel.style.cssText =
      'width:min(1280px,calc(100vw - 36px));height:min(86vh,920px);background:rgba(255,255,255,.98);border:1px solid rgba(0,0,0,.25);border-radius:14px;box-shadow:0 20px 60px rgba(0,0,0,.35);display:flex;flex-direction:column;overflow:hidden;font:14px/1.45 -apple-system,BlinkMacSystemFont,Segoe UI,Roboto,Helvetica,Arial,sans-serif;color:#111;';

    // Header
    const header = document.createElement('div');
    header.style.cssText =
      'padding:14px 16px;border-bottom:1px solid rgba(0,0,0,.12);display:flex;align-items:center;gap:10px;';
    const title = document.createElement('div');
    title.style.cssText = 'font-weight:700;';
    title.textContent = `Article Jump (${items.length})`;
    const spacer = document.createElement('div');
    spacer.style.cssText = 'flex:1;';
    const close = document.createElement('button');
    close.type = 'button';
    close.textContent = 'Close';
    close.setAttribute('aria-label', 'Close');
    close.style.cssText =
      'border:1px solid rgba(0,0,0,.25);background:#fff;border-radius:10px;padding:8px 10px;cursor:pointer;';
    header.appendChild(title);
    header.appendChild(spacer);
    header.appendChild(close);

    // Body
    const body = document.createElement('div');
    body.style.cssText =
      'padding:14px 16px;display:flex;flex-direction:column;gap:12px;overflow:hidden;';

    // Combobox row
    const row = document.createElement('div');
    row.style.cssText = 'display:flex;gap:10px;align-items:center;';

    const comboWrap = document.createElement('div');
    comboWrap.style.cssText = 'position:relative;flex:1;min-width:0;';

    const input = document.createElement('input');
    input.type = 'text';
    input.placeholder = 'Search target article…';
    input.autocomplete = 'off';
    input.spellcheck = false;
    input.style.cssText =
      'width:100%;box-sizing:border-box;padding:10px 12px;border-radius:12px;border:1px solid rgba(0,0,0,.25);background:#fff;font-size:14px;';

    const listId = KEY + '_listbox';
    input.setAttribute('role', 'combobox');
    input.setAttribute('aria-autocomplete', 'list');
    input.setAttribute('aria-expanded', 'false');
    input.setAttribute('aria-controls', listId);
    input.setAttribute('aria-haspopup', 'listbox');

    // Listbox: make it a LARGE, always-within-panel area instead of a small dropdown
    const list = document.createElement('div');
    list.id = listId;
    list.setAttribute('role', 'listbox');
    list.style.cssText =
      'margin-top:12px;flex:1;min-height:240px;overflow:auto;border:1px solid rgba(0,0,0,.18);border-radius:14px;background:#fff;box-shadow:0 12px 32px rgba(0,0,0,.12);';

    const go = document.createElement('button');
    go.type = 'button';
    go.textContent = 'Go';
    go.style.cssText =
      'padding:10px 14px;border-radius:12px;border:1px solid rgba(0,0,0,.25);background:#f6f6f6;cursor:pointer;white-space:nowrap;font-size:14px;';

    const hint = document.createElement('div');
    hint.style.cssText = 'color:rgba(0,0,0,.65);font-size:12.5px;';
    hint.textContent =
      'Type to filter. Use ↑/↓ to navigate, Enter to jump, Esc to close. Click outside the panel to close.';

    // Footer
    const footer = document.createElement('div');
    footer.style.cssText =
      'padding:12px 16px;border-top:1px solid rgba(0,0,0,.12);display:flex;gap:10px;align-items:center;justify-content:space-between;color:rgba(0,0,0,.7);font-size:12.5px;';
    const leftInfo = document.createElement('div');
    leftInfo.textContent = 'Enter: jump';
    const rightInfo = document.createElement('div');
    rightInfo.textContent = 'Esc: close';
    footer.appendChild(leftInfo);
    footer.appendChild(rightInfo);

    // Compose body
    row.appendChild(comboWrap);
    row.appendChild(go);
    comboWrap.appendChild(input);
    body.appendChild(row);
    body.appendChild(hint);
    body.appendChild(list);

    panel.appendChild(header);
    panel.appendChild(body);
    panel.appendChild(footer);
    overlay.appendChild(panel);
    document.documentElement.appendChild(overlay);

    // State
    let filtered = [...items];
    let activeIndex = -1;

    const optionId = (idx) => `${KEY}_opt_${idx}`;

    const highlight = (idx, scroll = true) => {
      const max = filtered.length - 1;
      const n = Math.max(-1, Math.min(idx, max));
      activeIndex = n;

      const opts = list.querySelectorAll('[role="option"]');
      opts.forEach((el, j) => {
        const active = j === activeIndex;
        el.setAttribute('aria-selected', active ? 'true' : 'false');
        el.style.background = active ? 'rgba(0,0,0,.06)' : 'transparent';
      });

      if (activeIndex >= 0) {
        const id = optionId(activeIndex);
        input.setAttribute('aria-activedescendant', id);
        if (scroll) {
          const el = document.getElementById(id);
          if (el) el.scrollIntoView({ block: 'nearest' });
        }
      } else {
        input.removeAttribute('aria-activedescendant');
      }
    };

    const jump = (item) => {
      if (!item) return;
      const a = item.el;

      cleanup();

      a.scrollIntoView({ behavior: 'smooth', block: 'start' });
      a.style.outline = '3px solid rgba(0,120,255,.55)';
      a.style.outlineOffset = '4px';
      setTimeout(() => {
        a.style.outline = '';
        a.style.outlineOffset = '';
      }, 1200);
    };

    const selectActive = () => {
      if (activeIndex < 0 || activeIndex >= filtered.length) return;
      const it = filtered[activeIndex];
      input.value = it.label;
      jump(it);
    };

    const render = () => {
      list.innerHTML = '';
      if (!filtered.length) {
        const empty = document.createElement('div');
        empty.style.cssText = 'padding:12px 12px;color:rgba(0,0,0,.6);';
        empty.textContent = 'No matches';
        list.appendChild(empty);
        highlight(-1, false);
        return;
      }

      filtered.forEach((it, j) => {
        const opt = document.createElement('div');
        opt.id = optionId(j);
        opt.setAttribute('role', 'option');
        opt.setAttribute('aria-selected', 'false');
        opt.style.cssText =
          'padding:10px 12px;cursor:pointer;user-select:none;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;';
        opt.textContent = it.label;

        opt.addEventListener('mousemove', () => highlight(j, false));
        opt.addEventListener('mousedown', (e) => e.preventDefault());
        opt.addEventListener('click', () => {
          highlight(j, false);
          selectActive();
        });

        list.appendChild(opt);
      });

      if (activeIndex >= filtered.length) activeIndex = -1;
      if (activeIndex === -1) highlight(0, false);
      else highlight(activeIndex, false);
    };

    const filterNow = () => {
      const q = normalize(input.value).toLowerCase();
      filtered = q
        ? items.filter((it) => it.label.toLowerCase().includes(q))
        : items.slice();
      activeIndex = -1;
      render();
    };

    const onOverlayDown = (e) => {
      if (e.target === overlay) cleanup();
    };

    const onKey = (e) => {
      if (e.key === 'Escape') {
        e.preventDefault();
        cleanup();
      }
    };

    const cleanup = () => {
      overlay.removeEventListener('mousedown', onOverlayDown, true);
      document.removeEventListener('keydown', onKey, true);
      overlay.remove();
    };

    close.onclick = cleanup;

    overlay.addEventListener('mousedown', onOverlayDown, true);
    document.addEventListener('keydown', onKey, true);

    input.addEventListener('input', () => {
      filterNow();
    });

    input.addEventListener('keydown', (e) => {
      if (e.key === 'ArrowDown') {
        e.preventDefault();
        if (!filtered.length) return;
        highlight(activeIndex + 1);
      } else if (e.key === 'ArrowUp') {
        e.preventDefault();
        if (!filtered.length) return;
        highlight(activeIndex - 1);
      } else if (e.key === 'Enter') {
        e.preventDefault();
        selectActive();
      } else if (e.key === 'Tab') {
        // keep modal open; do nothing special
      } else if (e.key === 'Escape') {
        e.preventDefault();
        cleanup();
      }
    });

    go.addEventListener('click', () => {
      if (activeIndex === -1 && filtered.length) highlight(0, false);
      selectActive();
    });

    // Initial render
    filterNow();
    input.focus();
  } catch (e) {
    alert('Bookmarklet error: ' + (e && e.message ? e.message : e));
  }
})();
```
