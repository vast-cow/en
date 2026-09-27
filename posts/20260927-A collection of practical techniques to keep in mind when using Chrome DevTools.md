---
pubDatetime: 2026-09-27T13:53:00+09:00
title: "A collection of practical techniques to keep in mind when using Chrome DevTools."
description: "Chrome DevTools offers numerous convenient features that can streamline debugging tasks related to the DOM, events, communication, and rendering. This document categorizes frequently used features, such as `inspect()`, `$0`, and `monitorEvents()`, based on their specific applications to facilitate everyday debugging."
---

Chrome DevTools has many useful features that can significantly improve your debugging efficiency.

Here are some of the most practical ones for everyday use:

## `inspect()`

This allows you to directly inspect a specified element in DevTools.

When you pass a DOM element, it will be selected in the Elements panel.

```javascript
inspect(document.querySelector('.button'))
inspect($0)
```

This is useful when you want to check an element you found in JavaScript directly in the Elements panel.

---

## `$0` / `$1` / `$2`

These allow you to reference elements selected in the Elements panel from the Console.

- `$0`: The currently selected element
- `$1`: The element selected before the current one
- `$2`: The element selected two steps before the current one

```javascript
$0
$0.textContent
$0.getBoundingClientRect()
```

This is helpful when you need to switch between the Elements and Console panels while investigating.

---

## `$()` / `$$()`

These are simplified selectors specifically for the DevTools Console.

- `$()`: Selects a single element, like `querySelector()`.
- `$$()`: Selects multiple elements, like `querySelectorAll()`.

```javascript
$('.button')
$$('a')
```

This saves you from having to type `document.querySelector()` every time you want to quickly check a DOM element.

---

## `monitorEvents()`

This displays events that occur on a specified element in the Console.

```javascript
monitorEvents($0, 'click')
monitorEvents($0, ['click', 'input', 'change'])
```

To stop monitoring events, use:

```javascript
unmonitorEvents($0)
```

This is useful when you want to find out what events are triggered by a specific action.

---

## `getEventListeners()`

This allows you to check the event listeners registered on a target element.

```javascript
getEventListeners($0)
```

This is useful when you want to see what events, such as `click` or `keydown`, are registered.

---

## `copy()`

This copies a value to the clipboard.

```javascript
copy($0.outerHTML)
copy(JSON.stringify(data, null, 2))
```

This is helpful when you want to quickly paste DOM elements or API responses into an editor or chat.

---

## `console.table()`

This displays arrays and objects in a tabular format.

```javascript
console.table(users)
```

When viewing an array of objects, this is much easier to read than a regular `console.log()`.

---

## `console.trace()`

This displays the stack trace, showing you where the current process was called from.

```javascript
console.trace()
```

This is useful when you want to investigate where a function is being called from.

---

## Break on DOM Changes

In the Elements panel, right-click on the target element and select `Break on` to specify the following conditions:

- Subtree modifications
- Attribute modifications
- Node removal

This is very powerful when you want to track who is modifying a specific DOM element.

---

## Event Listener Breakpoints

You can set these up in Sources → Event Listener Breakpoints.

You can pause JavaScript execution at the moment an event, such as `click`, `input`, or `keydown`, occurs.

This is useful when you don't know where the event handler is written.

---

## XHR / fetch Breakpoints

You can set these up in Sources → XHR/fetch Breakpoints.

By specifying a part of a URL, you can pause JavaScript execution at the moment that network request occurs.

This is useful when you want to find the code that is making a specific API call.

---

## Network: Preserve log

Enable `Preserve log` in the Network panel to retain network history even after page transitions or reloads.

This is helpful for investigating issues that span page transitions, such as login, OAuth, and redirects.

---

## Network: Copy as fetch / Copy as cURL

When you right-click on a request in the Network panel, you can use:

- Copy as fetch
- Copy as cURL

This is useful when you want to reproduce a request that occurred in the browser in code or a terminal.

---

## Request blocking

You can intentionally block specific JavaScript files, APIs, or images.

For example, you can use this to test:

- What happens to the screen if a specific API call fails?
- What happens if a specific JavaScript file cannot be loaded?

---

## Local Overrides

You can replace JavaScript, CSS, and HTML files on the network with local versions, and the changes will persist after a reload.

This allows you to test:

- What happens if I modify this code?

without directly modifying the code in the production environment.

---

## Coverage

Open the Command Menu:

```text
Ctrl/Cmd + Shift + P
```

and run:

```
Show Coverage
```

This will show you how much of the loaded CSS and JavaScript is actually being used.

This is useful when investigating unused CSS or large JavaScript bundles.

---

## Performance monitor

Open the Command Menu and run:

```
Show Performance monitor
```

This will display:

- CPU usage
- JS heap
- Number of DOM nodes
- Number of event listeners
- Layout / Style recalculation

in real-time.

This can be used to investigate issues such as whether the DOM or memory is continuously increasing during an operation.

---

## Rendering

Open the Command Menu and select `Show Rendering` to access rendering-related debugging features.

For example:

- Paint flashing
- Layout Shift Regions
- FPS meter
- CSS media feature emulation

This is useful when investigating areas with frequent repaints, layout shifts, or rendering performance issues.

---

## 7 Features to Remember First

The following 7 features are used most frequently:

```javascript
inspect(el)
$0
$$()
monitorEvents()
getEventListeners()
copy()
console.table()
```

You don't need to memorize everything.

Start by using `$0`, `$$()`, and `inspect()`, and you'll find that debugging DOM elements in Chrome DevTools becomes much easier.
