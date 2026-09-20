---
title: "Setting Reasoning Strength in OpenWebUI with `chat_template_kwargs`"
description: "When you run a model through llama.cpp and access it from OpenWebUI using an OpenAI-compatible API,..."
pubDatetime: 2026-01-20T02:50:16.813Z
---

When you run a model through **llama.cpp** and access it from **OpenWebUI** using an **OpenAI-compatible API**, you may want to control how “strongly” the model reasons. A reliable way to do this is to send a custom parameter called **`chat_template_kwargs`** from OpenWebUI. This parameter can include a **`reasoning_effort`** setting such as `low`, `medium`, or `high`.

## Why use `chat_template_kwargs`?

In many llama.cpp-based deployments, the model’s reasoning behavior is influenced by values passed into the chat template. Rather than trying to force reasoning strength through prompts, passing `reasoning_effort` via `chat_template_kwargs` provides a more direct and predictable control mechanism. OpenWebUI supports sending such custom parameters in its model configuration, and this approach is also demonstrated in official integration guidance (in a different backend example). [OpenVINO Documentation][2]

## How to set it in OpenWebUI

To configure reasoning strength for a specific model in OpenWebUI, follow these steps:

1. Go to **Admin Panel → Settings → Models**
2. Select the model you want to configure and open **Advanced Params**
3. Click **+ Add Custom Parameter**
4. Set:

   * **Parameter name:** `chat_template_kwargs`
   * **Value:** `{"reasoning_effort": "high"}`
     (You can change `"high"` to `"medium"` or `"low"` depending on your needs.)

Once saved, OpenWebUI will include this parameter in requests sent to your llama.cpp OpenAI-compatible endpoint. That makes the configuration apply consistently, without requiring users to manually adjust prompts each time they start a new chat.

## Choosing the right level

* **low**: Faster responses and less deep multi-step reasoning
* **medium**: Balanced performance and reasoning depth
* **high**: More thorough reasoning, often slower and more verbose internally

In practice, you should start with `medium` and move to `high` only when you truly need deeper reasoning for complex tasks.



[2]: https://docs.openvino.ai/2025/model-server/ovms_demos_integration_with_open_webui.html?utm_source=chatgpt.com "Demonstrating integration of Open WebUI with OpenVINO ..."
