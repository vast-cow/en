---
title: "A Simple “Select or Free Input” Web Component"
description: "Overview   This HTML page demonstrates a small user interface pattern that lets people..."
pubDatetime: 2026-02-05T13:12:08.073Z
---

## Overview

This HTML page demonstrates a small user interface pattern that lets people choose from a predefined list (a dropdown) or type any custom value (a text input). The component keeps a single “current value” and shows it on the page, regardless of whether it came from the dropdown or from free typing.

## Page Structure

The page includes a heading, a compact row containing the controls, and an output area:

* **Heading:** “Select or Free Input”
* **Control row:** a `<select>`, a hidden `<input>`, and a toggle `<button>`
* **Output line:** displays the current value inside a `<span>`

The dropdown starts with a placeholder option (`-- Please select --`) and includes three fruit options: Apple, Orange, and Banana. It also contains a special hidden option used only when the current value is custom (not part of the standard list).

## Styling

A few CSS rules keep the layout clean:

* `.row` uses flexbox with spacing and vertical alignment.
* `.hidden` sets `display: none` to hide elements without removing them from the DOM.
* Form elements and the button have consistent padding and font size.
* The output area uses a monospace font for a “developer console” feel.

## Core Behavior

The JavaScript implements a simple state machine with two pieces of state:

* `mode`: either `"select"` or `"input"`
* `value`: the current chosen/typed value (a string)

The UI always reflects this state:

* In **select mode**, the dropdown is visible and the text input is hidden.
* In **input mode**, the text input is visible and the dropdown is hidden.
* The button text changes to match the current mode:

  * “Free input” when you are in select mode
  * “Back to select” when you are in input mode
* The displayed output (`value: ...`) always shows the current `value`.

## Handling Custom Values in the Dropdown

A key detail is how the dropdown can display a custom typed value even though it is not one of the predefined options.

### Detecting Whether a Value Exists

A helper function checks whether the dropdown already contains an option with a given value by scanning `selectEl.options`.

### Maintaining a Hidden “Custom” Option

When the current value is **empty** or **matches an existing option**, the special custom option stays hidden.

When the current value is **not in the list**, the script:

* makes the custom option visible,
* sets its text to the actual typed value (so the dropdown shows the exact custom text),
* selects this custom option internally using the placeholder value `__custom__`.

This approach allows the dropdown to reflect any free-typed value without permanently adding it to the fixed list of options.

## Event Flow

The component updates state and UI through event listeners:

### Dropdown changes

When the user selects Apple/Orange/Banana (or the empty placeholder), the script updates `state.value` and re-renders. If the selected option is the special `__custom__`, it does not overwrite the value; it just keeps the existing custom value.

### Typing in the text input

As the user types, `state.value` updates live. The output display updates immediately, and the custom option is synchronized so the dropdown can later show that typed value.

### Committing typed input

When the input loses focus (`blur`), the typed value is “committed”:

* the current input text becomes the selected value,
* the component returns to select mode,
* the dropdown then shows either a matching predefined option or the custom option.

### Toggling between modes

Clicking the button switches the UI:

* From **select → input**: shows the text field, focuses it, and places the cursor at the end.
* From **input → select**: commits the current text (same behavior as blur) and returns to select mode.

## Why This Pattern Is Useful

This design is a practical “combo input” pattern:

* Users can pick quickly from common options.
* Users can still enter uncommon values without being blocked.
* The component keeps a consistent single source of truth (`state.value`).
* The dropdown can represent custom entries without modifying the original option set permanently.

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Select or Free Input Component</title>
  <style>
    .row { display: flex; gap: 8px; align-items: center; }
    .hidden { display: none; }
    select, input { padding: 6px 8px; font-size: 14px; }
    button { padding: 6px 10px; font-size: 14px; cursor: pointer; }
    .out { margin-top: 10px; font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace; }
  </style>
</head>
<body>
  <h3>Select or Free Input</h3>

  <div class="row" id="combo">
    <select id="comboSelect" aria-label="Choose from options">
      <option value="">-- Please select --</option>
      <option value="Apple">Apple</option>
      <option value="Orange">Orange</option>
      <option value="Banana">Banana</option>

      <!-- Option used to display a custom value (not in the predefined list) -->
      <option value="__custom__" id="customOpt" class="hidden"></option>
    </select>

    <input id="comboInput" class="hidden" type="text" placeholder="Type freely..." aria-label="Free input" />

    <button id="toggleBtn" type="button">Free input</button>
  </div>

  <div class="out">
    value: <span id="valueOut"></span>
  </div>

  <script>
    (() => {
      const selectEl = document.getElementById('comboSelect');
      const inputEl  = document.getElementById('comboInput');
      const toggleBtn = document.getElementById('toggleBtn');
      const valueOut = document.getElementById('valueOut');
      const customOpt = document.getElementById('customOpt');

      const state = { mode: 'select', value: '' };

      const hasOptionValue = (v) =>
        Array.from(selectEl.options).some(opt => opt.value === v);

      const syncCustomOption = () => {
        const v = state.value;

        // No custom option needed when empty or matching an existing option.
        if (!v || hasOptionValue(v)) {
          customOpt.classList.add('hidden');
          customOpt.textContent = '';
          return;
        }

        // Display the actual typed value (instead of "(Custom)").
        customOpt.textContent = v;
        customOpt.classList.remove('hidden');
      };

      const render = () => {
        const isSelect = state.mode === 'select';

        selectEl.classList.toggle('hidden', !isSelect);
        inputEl.classList.toggle('hidden', isSelect);

        toggleBtn.textContent = isSelect ? 'Free input' : 'Back to select';
        valueOut.textContent = state.value;

        syncCustomOption();

        if (isSelect) {
          if (state.value === '') {
            selectEl.value = '';
          } else if (hasOptionValue(state.value)) {
            selectEl.value = state.value;
          } else {
            // Not in list => choose the custom option.
            selectEl.value = '__custom__';
          }
        } else {
          inputEl.value = state.value;
        }
      };

      const commitInputValue = () => {
        // Commit the current text as the selected value (add trim() here if desired).
        state.value = inputEl.value;
        state.mode = 'select'; // Treat it as "selected" and return to select UI.
        render();
      };

      // Select changes
      selectEl.addEventListener('change', () => {
        if (selectEl.value !== '__custom__') {
          state.value = selectEl.value;
        }
        render();
      });

      // Live input updates
      inputEl.addEventListener('input', () => {
        state.value = inputEl.value;
        valueOut.textContent = state.value;
        syncCustomOption();
      });

      // Commit on blur (focus out)
      inputEl.addEventListener('blur', () => {
        commitInputValue();
      });

      // Toggle button
      toggleBtn.addEventListener('click', () => {
        if (state.mode === 'select') {
          state.mode = 'input';
          render();
          inputEl.focus();
          inputEl.setSelectionRange(inputEl.value.length, inputEl.value.length);
        } else {
          // Also commit when switching back via button (consistent with blur).
          commitInputValue();
        }
      });

      render();
    })();
  </script>
</body>
</html>
```
