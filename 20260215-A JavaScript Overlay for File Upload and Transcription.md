---
title: "A JavaScript Overlay for File Upload and Transcription"
description: "Upload a selected file to ChatGPT's transcription endpoint through a browser overlay that uses the current session token, detects audio duration, and displays text or API errors."
pubDatetime: 2026-02-15T14:48:58.003Z
updatedDate: 2026-02-15T15:03:06.357Z
---

This script is an immediately invoked async function that creates a browser-based overlay for uploading files to a transcription endpoint. It operates in strict mode and communicates with a backend API using a bearer access token extracted from the page.

## Core Functionality

The script dynamically builds a modal interface and handles secure file submission:

* Retrieves an `accessToken` from a JSON payload inside the `#client-bootstrap` element.
* Generates a full-screen overlay with file input, status display, and result panel.
* Optionally calculates audio duration (in milliseconds) using an HTMLAudioElement.
* Sends the selected file via `fetch` to a transcription endpoint using `FormData`.
* Displays the returned `json.text` field alongside the raw JSON response.

Error handling is implemented at each critical step, including token parsing, upload failure, and JSON extraction. The interface supports dismissal via Escape key, overlay click, or a close button. Overall, the script provides a self-contained client-side upload and transcription utility.

```javascript
(async () => { 
  "use strict"; 
 
  // ---- Config ---- 
  const UPLOAD_PATH = "https://chatgpt.com/backend-api/transcribe"; 
  const ACCEPT = "*/*"; // Accept any file type 
 
  // ---- Helpers ---- 
  const getAccessToken = () => { 
    const bootstrapEl = document.getElementById("client-bootstrap"); 
    if (!bootstrapEl) { 
      throw new Error('Element #client-bootstrap not found. Cannot retrieve access token.'); 
    } 
    let data; 
    try { 
      data = JSON.parse(bootstrapEl.innerHTML); 
    } catch (e) { 
      throw new Error("Failed to parse JSON from #client-bootstrap."); 
    } 
    const token = data?.session?.accessToken; 
    if (!token) { 
      throw new Error("accessToken not found. Check data.session.accessToken."); 
    } 
    return token; 
  }; 
 
  const formatMsFromAudioFile = async (file) => { 
    return await new Promise((resolve) => { 
      try { 
        const url = URL.createObjectURL(file); 
        const audio = document.createElement("audio"); 
        audio.preload = "metadata"; 
        audio.src = url; 
        audio.onloadedmetadata = () => { 
          URL.revokeObjectURL(url); 
          const durSec = audio.duration; 
          if (!Number.isFinite(durSec) || durSec <= 0) return resolve(null); 
          resolve(String(durSec * 1000)); 
        }; 
        audio.onerror = () => { 
          URL.revokeObjectURL(url); 
          resolve(null); 
        }; 
      } catch { 
        resolve(null); 
      } 
    }); 
  }; 
 
  const createOverlay = () => { 
    const overlay = document.createElement("div"); 
    overlay.id = "__webm_upload_overlay__"; 
    overlay.style.position = "fixed"; 
    overlay.style.inset = "0"; 
    overlay.style.zIndex = "2147483647"; 
    overlay.style.background = "#ffffff"; 
    overlay.style.display = "flex"; 
    overlay.style.alignItems = "center"; 
    overlay.style.justifyContent = "center"; 
    overlay.style.padding = "24px"; 
 
    const panel = document.createElement("div"); 
    panel.style.width = "min(720px, 92vw)"; 
    panel.style.background = "#ffffff"; 
    panel.style.color = "#111"; 
    panel.style.border = "1px solid rgba(0,0,0,0.15)"; 
    panel.style.borderRadius = "12px"; 
    panel.style.boxShadow = "0 12px 48px rgba(0,0,0,0.15)"; 
    panel.style.padding = "16px"; 
    panel.style.fontFamily = "ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial"; 
    panel.style.lineHeight = "1.4"; 
 
    const title = document.createElement("div"); 
    title.textContent = "Select and upload a file"; 
    title.style.fontSize = "16px"; 
    title.style.fontWeight = "600"; 
    title.style.marginBottom = "10px"; 
 
    const desc = document.createElement("div"); 
    desc.textContent = "Select a file → Send. The response JSON text field will be displayed."; 
    desc.style.opacity = "0.85"; 
    desc.style.marginBottom = "12px"; 
 
    const fileRow = document.createElement("div"); 
    fileRow.style.display = "flex"; 
    fileRow.style.gap = "10px"; 
    fileRow.style.alignItems = "center"; 
    fileRow.style.marginBottom = "12px"; 
 
    const fileInput = document.createElement("input"); 
    fileInput.type = "file"; 
    fileInput.accept = ACCEPT; 
    fileInput.style.flex = "1"; 
    fileInput.style.padding = "8px"; 
    fileInput.style.background = "#ffffff"; 
    fileInput.style.color = "#111"; 
    fileInput.style.border = "1px solid rgba(0,0,0,0.2)"; 
    fileInput.style.borderRadius = "8px"; 
 
    const sendBtn = document.createElement("button"); 
    sendBtn.textContent = "Send"; 
    sendBtn.disabled = true; 
    sendBtn.style.padding = "8px 12px"; 
    sendBtn.style.borderRadius = "8px"; 
    sendBtn.style.border = "1px solid rgba(0,0,0,0.2)"; 
    sendBtn.style.background = "#f0f0f0"; 
    sendBtn.style.color = "#111"; 
    sendBtn.style.cursor = "pointer"; 
 
    const cancelBtn = document.createElement("button"); 
    cancelBtn.textContent = "Close"; 
    cancelBtn.style.padding = "8px 12px"; 
    cancelBtn.style.borderRadius = "8px"; 
    cancelBtn.style.border = "1px solid rgba(0,0,0,0.2)"; 
    cancelBtn.style.background = "transparent"; 
    cancelBtn.style.color = "#111"; 
    cancelBtn.style.cursor = "pointer"; 
 
    const btnRow = document.createElement("div"); 
    btnRow.style.display = "flex"; 
    btnRow.style.gap = "10px"; 
    btnRow.style.justifyContent = "flex-end"; 
    btnRow.style.marginTop = "8px"; 
 
    const status = document.createElement("div"); 
    status.style.marginTop = "12px"; 
    status.style.padding = "10px"; 
    status.style.background = "#f5f5f5"; 
    status.style.border = "1px solid rgba(0,0,0,0.12)"; 
    status.style.borderRadius = "10px"; 
    status.style.whiteSpace = "pre-wrap"; 
    status.style.wordBreak = "break-word"; 
    status.style.maxHeight = "150px"; 
    status.style.overflowY = "auto"; 
    status.textContent = "No file selected"; 
 
    const result = document.createElement("div"); 
    result.style.marginTop = "12px"; 
    result.style.padding = "10px"; 
    result.style.background = "#f5f5f5"; 
    result.style.border = "1px solid rgba(0,0,0,0.12)"; 
    result.style.borderRadius = "10px"; 
    result.style.whiteSpace = "pre-wrap"; 
    result.style.wordBreak = "break-word"; 
    result.style.maxHeight = "300px"; 
    result.style.overflowY = "auto"; 
    result.textContent = ""; 
 
    const close = () => { 
      overlay.remove(); 
      window.removeEventListener("keydown", onKeyDown, true); 
    }; 
 
    const onKeyDown = (e) => { 
      if (e.key === "Escape") close(); 
    }; 
 
    overlay.addEventListener("click", (e) => { 
      if (e.target === overlay) close(); 
    }); 
 
    cancelBtn.addEventListener("click", close); 
    window.addEventListener("keydown", onKeyDown, true); 
 
    fileInput.addEventListener("change", () => { 
      const f = fileInput.files?.[0]; 
      if (!f) { 
        sendBtn.disabled = true; 
        status.textContent = "No file selected"; 
        return; 
      } 
      sendBtn.disabled = false; 
      status.textContent = `Selected: ${f.name} (${Math.round(f.size / 1024)} KB)`; 
      result.textContent = ""; 
    }); 
 
    panel.appendChild(title); 
    panel.appendChild(desc); 
    fileRow.appendChild(fileInput); 
    fileRow.appendChild(sendBtn); 
    panel.appendChild(fileRow); 
    btnRow.appendChild(cancelBtn); 
    panel.appendChild(btnRow); 
    panel.appendChild(status); 
    panel.appendChild(result); 
    overlay.appendChild(panel); 
    document.documentElement.appendChild(overlay); 
 
    return { overlay, fileInput, sendBtn, status, result, close }; 
  }; 
 
  const existing = document.getElementById("__webm_upload_overlay__"); 
  if (existing) existing.remove(); 
 
  const ui = createOverlay(); 
 
  ui.sendBtn.addEventListener("click", async () => { 
    const file = ui.fileInput.files?.[0]; 
    if (!file) return; 
 
    ui.sendBtn.disabled = true; 
    ui.status.textContent = "Uploading..."; 
 
    try { 
      const accessToken = getAccessToken(); 
 
      const fd = new FormData(); 
      fd.append("file", file, file.name); 
 
      const durationMs = await formatMsFromAudioFile(file); 
      if (durationMs != null) { 
        fd.append("duration_ms", durationMs); 
      } 
 
      const res = await fetch(UPLOAD_PATH, { 
        method: "POST", 
        headers: { 
          authorization: "Bearer " + accessToken, 
        }, 
        body: fd, 
      }); 
 
      if (!res.ok) { 
        const bodyText = await res.text().catch(() => ""); 
        throw new Error(`HTTP ${res.status} ${res.statusText}\n${bodyText}`); 
      } 
 
      const json = await res.json(); 
      const text = json?.text; 
 
      ui.status.textContent = "Completed"; 
      ui.result.textContent = 
        (typeof text === "string" ? text : "(json.text not found)") + 
        "\n\n--- raw json ---\n" + 
        JSON.stringify(json, null, 2); 
    } catch (e) { 
      ui.status.textContent = "Error"; 
      ui.result.textContent = String(e?.message || e); 
    } finally { 
      const f = ui.fileInput.files?.[0]; 
      ui.sendBtn.disabled = !f; 
    } 
  }); 
})(); 
```
