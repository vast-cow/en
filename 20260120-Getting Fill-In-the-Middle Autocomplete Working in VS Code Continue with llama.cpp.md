---
title: "Getting Fill-In-the-Middle Autocomplete Working in VS Code Continue with llama.cpp"
description: "Overview   Continue is a popular AI coding extension for Visual Studio Code. One of its most..."
pubDatetime: 2026-01-20T02:55:41.180Z
---

## Overview

Continue is a popular AI coding extension for Visual Studio Code. One of its most useful capabilities is **Tab autocomplete**, which is typically implemented as **Fill-In-the-Middle (FIM)** completion: the model predicts code that fits between what you already have before and after the cursor.

Community reports indicate that **llama.cpp (via `llama-server`) can be a practical backend for Continue’s FIM-style autocomplete**, often with better results than other local backends in some setups. In Continue’s configuration, the key idea is to define a model that is explicitly assigned the **`autocomplete` role**, and point it to your running `llama-server`.

## How Continue Chooses an Autocomplete Model

Continue’s YAML configuration supports multiple models. Each model can be assigned one or more **roles** (for example: `chat`, `edit`, `apply`, and importantly, `autocomplete`). When you include a model with the `autocomplete` role, Continue will use it for Tab completion and similar inline suggestions.

This is the modern equivalent of the older “tab autocomplete model” concept discussed in issues and community threads: you dedicate a model specifically for completion, rather than using your chat model for everything.

## Example `config.yaml` (Simple and Practical)

Below is a minimal, workable pattern: one model for FIM autocomplete via llama.cpp, and optionally another for chat/edit.

```yaml
name: Local Continue (llama.cpp FIM)
version: 1.0.0
schema: v1

models:
  # Autocomplete (FIM) model powered by llama.cpp / llama-server
  - name: local-fim
    provider: llama.cpp
    model: your-model.gguf
    apiBase: http://127.0.0.1:8081
    roles:
      - autocomplete
    autocompleteOptions:
      debounceDelay: 250
      maxPromptTokens: 1024
      modelTimeout: 2000

  # Optional: separate model for chat/edit/apply (OpenAI-compatible example)
  - name: chat-main
    provider: openai
    model: your-chat-model
    apiBase: http://127.0.0.1:9000/v1
    roles:
      - chat
      - edit
      - apply
```

Key points:

* **`provider: llama.cpp`** tells Continue to treat this model as a llama.cpp backend.
* **`apiBase`** should match where your `llama-server` is listening (example: `http://127.0.0.1:8081`).
* **`roles: [autocomplete]`** is what makes this model the one Continue uses for FIM-style Tab completion.

## Notes from Community Experience

A recurring theme in user reports is that **llama.cpp is often chosen specifically for autocomplete**, while a different backend may be used for chat, depending on performance and quality preferences. Users also discuss model sizing and latency tradeoffs (for example, using smaller “coder” models to keep suggestions responsive).

Separately, Continue’s own issue discussions confirm that the extension includes user-facing options related to “FIM” and “Next Edit autocomplete over FIM,” reinforcing that FIM is a core part of its autocomplete pathway.

## Troubleshooting Checklist

If autocomplete does not appear to work:

1. Confirm `llama-server` is running and reachable at the configured `apiBase`.
2. Ensure your FIM model entry includes `roles: [autocomplete]`.
3. Use a model known to behave well for code completions (and small enough to keep latency low).
4. If you are also running an OpenAI-compatible server for chat, keep it separate from llama.cpp autocomplete to avoid endpoint mismatches and confusion in routing.

## Conclusion

To enable FIM-style code completion in Continue using llama.cpp, you do not need an elaborate setup: **define a dedicated model entry with `provider: llama.cpp` and `roles: [autocomplete]`, and point it to your running `llama-server`.** From there, you can optionally layer on a separate chat/edit model depending on your workflow.
