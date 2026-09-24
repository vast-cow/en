---
title: "A Simple Bookmarklet Selector UI"
description: "Combine multiple bookmarklets in a searchable modal menu with keyboard navigation, ARIA attributes, and cleanup that restores the page before running a selection."
pubDatetime: 2026-01-19T08:38:58.571Z
---

This article explains a JavaScript bookmarklet that provides a user interface for selecting and running other bookmarklets. The goal of this script is to make it easier to manage and execute multiple bookmarklets from a single, accessible menu.

## Overview

The bookmarklet creates a modal dialog that appears on top of the current web page. Inside this dialog, users can search, navigate, and run predefined bookmarklets. The entire interface is built dynamically using plain JavaScript and does not rely on external libraries.

## Embedded Bookmarklets

At the core of the script is a list of embedded bookmarklets. Each bookmarklet is defined by a title and a function. For example, one sample bookmarklet shows an alert, and another displays the current page URL. Additional bookmarklets can be added by extending this list.

## User Interface Structure

### Overlay and Modal Dialog

When the bookmarklet runs, it creates a full-screen overlay that darkens the background and prevents scrolling. A centered modal dialog is displayed on top of this overlay. The dialog uses a clean, modern design with rounded corners and clear visual separation between sections.

### Header

The header displays the title “Bookmarklet Selector” and a short usage hint. It also includes a close button that allows the user to dismiss the dialog at any time.

### Body

The body contains a search input and a list of available bookmarklets. The input acts as a combobox, allowing users to filter bookmarklets by typing. The list updates in real time to show only matching items.

### Footer

The footer provides two buttons: “Run” to execute the selected bookmarklet, and “Cancel” to close the dialog without running anything.

## Interaction and Accessibility

The script supports both mouse and keyboard interaction. Users can navigate the list with the up and down arrow keys, run a bookmarklet with the Enter key, and close the dialog with the Escape key. ARIA roles and attributes are applied to improve accessibility for assistive technologies.

## Execution and Cleanup

When a bookmarklet is selected and run, the dialog is removed and the original page state is restored, including scroll behavior. The selected bookmarklet function is then executed safely within a try-catch block to handle errors gracefully.

## Conclusion

This bookmarklet selector provides a practical and user-friendly way to organize and run multiple bookmarklets. By combining a searchable interface, keyboard support, and accessibility features, it improves the overall usability of bookmarklets for everyday browsing tasks.

```javascript
!(function () {
  'use strict';
  var items = [
    {
      title: 'Sample: Alert',
      code: function () {
        alert('hello');
      },
    },
    {
      title: 'Sample: Show URL',
      code: function () {
        alert(location.href);
      },
    },
  ];
  var ROOT_ID = 'bookmarklet-modal-aria';
  var old = document.getElementById(ROOT_ID);
  old && old.remove();
  var prevOverflow = document.documentElement.style.overflow;
  document.documentElement.style.overflow = 'hidden';
  var overlay = document.createElement('div');
  overlay.id = ROOT_ID;
  overlay.style.position = 'fixed';
  overlay.style.inset = '0';
  overlay.style.zIndex = '2147483647';
  overlay.style.background = 'rgba(0,0,0,0.6)';
  overlay.style.display = 'flex';
  overlay.style.alignItems = 'center';
  overlay.style.justifyContent = 'center';
  overlay.style.padding = '24px';
  overlay.style.boxSizing = 'border-box';
  var dialog = document.createElement('div');
  dialog.setAttribute('role', 'dialog');
  dialog.setAttribute('aria-modal', 'true');
  dialog.style.width = 'min(720px, 100%)';
  dialog.style.maxHeight = 'min(560px, 100%)';
  dialog.style.background = 'rgba(20,20,20,0.96)';
  dialog.style.color = '#fff';
  dialog.style.borderRadius = '16px';
  dialog.style.boxShadow = '0 20px 60px rgba(0,0,0,0.5)';
  dialog.style.border = '1px solid rgba(255,255,255,0.14)';
  dialog.style.display = 'flex';
  dialog.style.flexDirection = 'column';
  dialog.style.overflow = 'hidden';
  dialog.style.font =
    '14px/1.45 -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,Helvetica,Arial,sans-serif';
  overlay.appendChild(dialog);
  var header = document.createElement('div');
  header.style.padding = '16px 18px 12px';
  header.style.borderBottom = '1px solid rgba(255,255,255,0.12)';
  header.style.display = 'flex';
  header.style.justifyContent = 'space-between';
  header.style.alignItems = 'center';
  var titleWrap = document.createElement('div');
  var title = document.createElement('div');
  title.textContent = 'Bookmarklet Selector';
  title.style.fontSize = '16px';
  title.style.fontWeight = '700';
  var hint = document.createElement('div');
  hint.textContent = 'Use ↑ ↓ to navigate · Enter to run · Esc to close';
  hint.style.fontSize = '12px';
  hint.style.opacity = '0.8';
  titleWrap.appendChild(title);
  titleWrap.appendChild(hint);
  var closeTop = document.createElement('button');
  closeTop.textContent = 'Close';
  closeTop.type = 'button';
  closeTop.style.padding = '8px 12px';
  closeTop.style.borderRadius = '10px';
  closeTop.style.border = '1px solid rgba(255,255,255,0.18)';
  closeTop.style.background = 'rgba(255,255,255,0.10)';
  closeTop.style.color = '#fff';
  closeTop.style.cursor = 'pointer';
  header.appendChild(titleWrap);
  header.appendChild(closeTop);
  dialog.appendChild(header);
  var body = document.createElement('div');
  body.style.padding = '16px 18px';
  body.style.display = 'flex';
  body.style.flexDirection = 'column';
  body.style.gap = '12px';
  dialog.appendChild(body);
  var uid = Date.now() + '-' + Math.floor(1e5 * Math.random());
  var listboxId = 'bookmarklet-listbox-' + uid;
  var label = document.createElement('div');
  label.textContent = 'Search and select';
  label.style.fontSize = '12px';
  label.style.opacity = '0.9';
  body.appendChild(label);
  var input = document.createElement('input');
  input.type = 'text';
  input.placeholder = 'Type to filter bookmarklets';
  input.setAttribute('role', 'combobox');
  input.setAttribute('aria-autocomplete', 'list');
  input.setAttribute('aria-expanded', 'true');
  input.setAttribute('aria-controls', listboxId);
  input.setAttribute('aria-haspopup', 'listbox');
  input.style.width = '100%';
  input.style.padding = '10px 12px';
  input.style.borderRadius = '10px';
  input.style.border = '1px solid rgba(255,255,255,0.18)';
  input.style.background = 'rgba(255,255,255,0.10)';
  input.style.color = '#fff';
  input.style.outline = 'none';
  body.appendChild(input);
  var listbox = document.createElement('div');
  listbox.id = listboxId;
  listbox.setAttribute('role', 'listbox');
  listbox.style.maxHeight = '280px';
  listbox.style.overflow = 'auto';
  listbox.style.border = '1px solid rgba(255,255,255,0.18)';
  listbox.style.borderRadius = '12px';
  listbox.style.background = 'rgba(255,255,255,0.06)';
  listbox.style.padding = '6px';
  body.appendChild(listbox);
  var footer = document.createElement('div');
  footer.style.padding = '12px 18px 16px';
  footer.style.borderTop = '1px solid rgba(255,255,255,0.12)';
  footer.style.display = 'flex';
  footer.style.gap = '10px';
  function makeButton(label, primary) {
    var b = document.createElement('button');
    b.type = 'button';
    b.textContent = label;
    b.style.flex = '1';
    b.style.padding = '10px 12px';
    b.style.borderRadius = '12px';
    b.style.border = '1px solid rgba(255,255,255,0.18)';
    b.style.background = primary
      ? 'rgba(255,255,255,0.20)'
      : 'rgba(255,255,255,0.10)';
    b.style.color = '#fff';
    b.style.cursor = 'pointer';
    return b;
  }
  var runBtn = makeButton('Run', !0);
  var cancelBtn = makeButton('Cancel', !1);
  footer.appendChild(runBtn);
  footer.appendChild(cancelBtn);
  dialog.appendChild(footer);
  var filtered = items.map(function (it, i) {
    return { index: i, title: it.title };
  });
  var active = 0;
  var optionEls = [];
  function setActive(pos) {
    if (0 !== filtered.length) {
      active = Math.max(0, Math.min(pos, filtered.length - 1));
      optionEls.forEach(function (el, i) {
        el.setAttribute('aria-selected', i === active ? 'true' : 'false');
        el.style.background =
          i === active ? 'rgba(255,255,255,0.18)' : 'transparent';
      });
      var el = optionEls[active];
      if (el) {
        input.setAttribute('aria-activedescendant', el.id);
        el.scrollIntoView({ block: 'nearest' });
      }
    }
  }
  function render() {
    listbox.innerHTML = '';
    optionEls = [];
    if (filtered.length) {
      filtered.forEach(function (item, pos) {
        var opt = document.createElement('div');
        opt.id = 'opt-' + uid + '-' + pos;
        opt.setAttribute('role', 'option');
        opt.textContent = item.title;
        opt.style.padding = '10px';
        opt.style.borderRadius = '10px';
        opt.style.cursor = 'pointer';
        opt.onmouseenter = function () {
          setActive(pos);
        };
        opt.onclick = function () {
          setActive(pos);
          runSelected();
        };
        listbox.appendChild(opt);
        optionEls.push(opt);
      });
      setActive(active);
    } else {
      var empty = document.createElement('div');
      empty.textContent = 'No matching items';
      empty.style.padding = '10px';
      empty.style.opacity = '0.8';
      listbox.appendChild(empty);
    }
  }
  function cleanup() {
    document.documentElement.style.overflow = prevOverflow;
    document.removeEventListener('keydown', onKey, !0);
    overlay.remove();
  }
  function runSelected() {
    if (filtered.length) {
      var item = items[filtered[active].index];
      cleanup();
      try {
        item.code.call(window);
      } catch (e) {
        alert('Bookmarklet error: ' + e.message);
      }
    }
  }
  function onKey(e) {
    if ('Escape' === e.key) {
      e.preventDefault();
      cleanup();
    } else if ('Enter' === e.key) {
      e.preventDefault();
      runSelected();
    } else if ('ArrowDown' === e.key) {
      e.preventDefault();
      setActive(active + 1);
    } else if ('ArrowUp' === e.key) {
      e.preventDefault();
      setActive(active - 1);
    }
  }
  input.oninput = function () {
    var q = input.value.toLowerCase();
    filtered = items
      .map(function (it, i) {
        return { index: i, title: it.title };
      })
      .filter(function (x) {
        return x.title.toLowerCase().includes(q);
      });
    active = 0;
    render();
  };
  runBtn.onclick = runSelected;
  cancelBtn.onclick = cleanup;
  closeTop.onclick = cleanup;
  document.addEventListener('keydown', onKey, !0);
  render();
  document.documentElement.appendChild(overlay);
  input.focus();
})();
```
