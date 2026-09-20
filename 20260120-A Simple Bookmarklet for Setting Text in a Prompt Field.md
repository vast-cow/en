---
title: "A Simple Bookmarklet for Setting Text in a Prompt Field"
description: "Overview   This JavaScript snippet is a bookmarklet designed to simplify the process of..."
pubDatetime: 2026-01-20T06:19:12.498Z
updatedDate: 2026-04-16T11:08:16.332Z
---

## Overview

This JavaScript snippet is a **bookmarklet** designed to simplify the process of inserting multi-line text into a specific input field on a webpage. Instead of manually typing or pasting content line by line, this tool opens a small helper window where you can enter your text and automatically transfer it to the target field.

Its primary purpose is convenience: it allows users to quickly structure and inject formatted text into a prompt area.

---

## What This Tool Does

When executed, the script performs the following actions:

* Opens a **new popup window** with a simple interface.
* Displays a **text area** where you can enter or paste content.
* Provides a **“Set” button** to apply the text.
* Inserts the text into a specific field (`#prompt-textarea`) in the original page.
* Automatically formats the input into **separate lines (paragraphs)**.

---

## How It Works (High-Level)

The bookmarklet operates in two main parts:

### 1. Creating the Input Window

* A temporary HTML page is generated using a `Blob`.
* This page contains:

  * A title (based on the current page)
  * A text area
  * A button to trigger the action
* The page is opened in a new browser window.

### 2. Sending Text Back to the Original Page

* When you click **Set**:

  * The script accesses the original (parent) window.
  * It finds the target element (`#prompt-textarea`).
  * Your input is split into lines.
  * Each line is converted into a paragraph element.
  * Existing content in the target field is replaced.
  * The view scrolls to the last inserted line.
  * The popup window closes automatically.

---

## How to Use It

### Step 1: Create the Bookmarklet

* Save the entire script as a bookmark in your browser.
* Ensure the URL starts with `javascript:`.

### Step 2: Open the Target Page

* Navigate to a page that contains a text input area with the ID:

  ```plaintext
  #prompt-textarea
  ```

### Step 3: Run the Bookmarklet

* Click the bookmarklet.
* A small popup window will appear.

### Step 4: Enter Your Text

* Type or paste your multi-line content into the text box.

### Step 5: Apply the Text

* Click the **Set** button.
* The content will be inserted into the original page automatically.

---

## Key Features

* **Multi-line support**: Preserves line breaks as separate paragraphs.
* **Automatic formatting**: Converts empty lines into spacing.
* **Quick replacement**: Clears existing content before inserting new text.
* **Minimal interface**: Simple and focused user experience.

---

## Limitations

* Works only if the original page contains an element with the ID `#prompt-textarea`.
* Requires popup windows to be allowed in the browser.
* Depends on the ability to access the parent window (same-origin restrictions may apply).

---

## Summary

This bookmarklet provides a lightweight way to input structured text into a webpage without manual formatting. It is especially useful for repetitive workflows where consistent text formatting is needed.

```javascript
javascript:(()=>{const escapedParentTitle=(document.title||%22Prompt Setter%22).replace(/&/g,%22&amp;%22).replace(/</g,%22&lt;%22).replace(/>/g,%22&gt;%22);const blob=new Blob([`<!doctype html>\n<html lang=%22en%22>\n<head>\n  <meta charset=%22utf-8%22>\n  <title>${escapedParentTitle}</title>\n  <style>\n    body {\n      font-family: sans-serif;\n      margin: 16px;\n    }\n    .page-title {\n      font-size: 24px;\n      font-weight: 700;\n      line-height: 1.4;\n      margin: 0 0 16px;\n      word-break: break-word;\n    }\n    textarea {\n      width: 100%25;\n      height: 240px;\n      box-sizing: border-box;\n      font-family: monospace;\n      font-size: 14px;\n    }\n    button {\n      margin-top: 12px;\n      padding: 8px 16px;\n      font-size: 14px;\n    }\n    .msg {\n      margin-top: 12px;\n      color: #333;\n%20%20%20%20%20%20white-space:%20pre-wrap;\n%20%20%20%20}\n%20%20%3C/style%3E\n%3C/head%3E\n%3Cbody%3E\n%20%20%3Cdiv%20class=%22page-title%22%3E${escapedParentTitle}%3C/div%3E\n%20%20%3Ctextarea%20id=%22usertext%22%20placeholder=%22Enter%20text%20here%22%3E%3C/textarea%3E\n%20%20%3Cbr%3E\n%20%20%3Cbutton%20id=%22setButton%22%20type=%22button%22%3ESet%3C/button%3E\n%20%20%3Cdiv%20class=%22msg%22%20id=%22msg%22%3E%3C/div%3E\n%3C/body%3E\n%3C/html%3E%60],{type:%22text/html%22});const%20url=URL.createObjectURL(blob);const%20child=window.open(url,%22_blank%22);if(!child){alert(%22Could%20not%20open%20popup%22);return}const%20setup=()=%3E{try{const%20childDoc=child.document;const%20usertextEl=childDoc.getElementById(%22usertext%22);const%20setButton=childDoc.getElementById(%22setButton%22);const%20msgEl=childDoc.getElementById(%22msg%22);if(!usertextEl||!setButton||!msgEl){setTimeout(setup,50);return}setButton.addEventListener(%22click%22,async()=%3E{try{if(!child.opener||child.opener.closed)throw%20new%20Error(%22Cannot%20access%20parent%20window%22);const%20openerDoc=child.opener.document;const%20promptTextarea=openerDoc.querySelector(%22#prompt-textarea%22);if(!promptTextarea)throw%20new%20Error(%22Could%20not%20find%20#prompt-textarea%22);const%20usertext=usertextEl.value;const%20promptInputList=usertext.replace(/\r\n/g,%22\n%22).split(%22\n%22);const%20childNodesSaved=Array.from(promptTextarea.childNodes);for(const%20promptLine%20of%20promptInputList){const%20p=openerDoc.createElement(%22p%22);%22%22===promptLine?p.appendChild(openerDoc.createElement(%22br%22)):p.textContent=promptLine;promptTextarea.appendChild(p)}for(const%20childNodeSaved%20of%20childNodesSaved)promptTextarea.removeChild(childNodeSaved);await%20new%20Promise(resolve=%3EsetTimeout(resolve,0));promptTextarea.lastChild&&promptTextarea.lastChild.scrollIntoView&&promptTextarea.lastChild.scrollIntoView();child.close()}catch(err){msgEl.textContent=%22Error:%20%22+err.message}})}catch(e){setTimeout(setup,50)}};setup()})();
```
