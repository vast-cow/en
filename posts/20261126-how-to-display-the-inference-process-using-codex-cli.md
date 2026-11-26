---
title: "How to display the inference process using the Codex CLI"
description: "This article explains how to control the display of inference results using the Codex CLI configuration file or startup options."
pubDatetime: 2026-11-26T10:44:00+09:00
---
To display the reasoning process in Codex CLI, edit `~/.codex/config.toml`.

## **1. Enable permanent display in the configuration file**

```
vim ~/.codex/config.toml
```

Add the following settings to the top level. If the same setting already exists, change its value.

```
hide_agent_reasoning = false
model_reasoning_summary = "detailed"
show_raw_agent_reasoning = true
```

Save the settings and restart Codex CLI.

| Setting                                   | Effect                   |
| -------------------------------------- | -------------------- |
| `hide_agent_reasoning = false`         | Does not hide reasoning events       |
| `model_reasoning_summary = "detailed"` | Requests a detailed reasoning summary       |
| `show_raw_agent_reasoning = true`      | Displays the raw reasoning content if the model returns it |

These are official Codex settings.

[image](https://www.google.com/s2/favicons?domain=https://github.com&sz=32)

GitHub+2

## **2. Enable temporarily with options when starting**

If you don't want to change the configuration file, you can specify it with the following command.

```
codex \
  -c 'hide_agent_reasoning=false' \
  -c 'model_reasoning_summary="detailed"' \
  -c 'show_raw_agent_reasoning=true'
```

## **3. Level of detail for the reasoning summary**

You can specify the following four types for `model_reasoning_summary`.

| Value          | Action          |
| ---------- | ----------- |
| `auto`     | Automatically selects based on the model |
| `concise`  | Concise reasoning summary   |
| `detailed` | Detailed reasoning summary   |
| `none`     | Disables reasoning summary  |

Also, if you want to improve the reasoning ability itself, add the following for supported models.

```
model_reasoning_effort = "high"
```

This is a setting independent of the display level of detail.

## **4. Notes**

**OpenAI models do not expose the internal thought process (Chain of Thought) itself.**

Even if you set `show_raw_agent_reasoning = true`, it will not be displayed unless the model returns the raw reasoning content. Normally, you can view only the reasoning summary.

[image](https://www.google.com/s2/favicons?domain=https://developers.openai.com&sz=32)

OpenAI API+1

Also, you can check the transcript by pressing `Ctrl + T` in the CLI.

Official documentation:

- [Configuration Reference](https://developers.openai.com/codex/config-reference?utm_source=chatgpt.com)
- [Advanced Configuration](https://developers.openai.com/codex/config-advanced?utm_source=chatgpt.com)
