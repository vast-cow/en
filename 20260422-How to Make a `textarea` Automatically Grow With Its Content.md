---
title: "How to Make a `textarea` Automatically Grow With Its Content"
description: "A textarea that expands as the user types makes forms feel much smoother. Instead of forcing users to..."
pubDatetime: 2026-04-22T14:35:04.571Z
---

A `textarea` that expands as the user types makes forms feel much smoother. Instead of forcing users to scroll inside the field, you can adjust its height automatically based on the amount of text.

A simple way to do this is to use JavaScript with `scrollHeight`.

### The idea

The pattern is straightforward:

1. Reset the height to `auto`
2. Set the height to the element’s `scrollHeight`

```js
function autoResize(textarea) {
  textarea.style.height = "auto";
  textarea.style.height = textarea.scrollHeight + "px";
}
```

This works because `scrollHeight` gives you the full height needed to display all of the content without an internal scrollbar.

### Run it on input

To keep the height in sync while the user types, call the function on every `input` event.

```js
const textarea = document.getElementById("message");

textarea.addEventListener("input", () => {
  autoResize(textarea);
});
```

### Recommended CSS

A few CSS rules help this behave cleanly:

```css
textarea {
  width: 100%;
  min-height: 120px;
  resize: none;
  overflow: hidden;
  box-sizing: border-box;
}
```

`resize: none` prevents manual dragging, and `overflow: hidden` avoids showing a scrollbar inside the field.

### Minimal example

Here is a complete, reusable version:

```html
<textarea id="message" placeholder="Type here..."></textarea>

<style>
  textarea {
    width: 100%;
    min-height: 120px;
    resize: none;
    overflow: hidden;
    box-sizing: border-box;
  }
</style>

<script>
  const textarea = document.getElementById("message");

  function autoResize(textarea) {
    textarea.style.height = "auto";
    textarea.style.height = textarea.scrollHeight + "px";
  }

  textarea.addEventListener("input", () => {
    autoResize(textarea);
  });

  autoResize(textarea);
</script>
```

### One small detail to remember

If you change the value with JavaScript, the height will not update by itself. After setting `textarea.value`, call `autoResize(textarea)` again.

In practice, that is all you need. By combining `scrollHeight` with a small resize function, you can add an auto-growing `textarea` to almost any page with minimal code.
