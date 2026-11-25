---
title: "When you encounter errors with MCP or image input in Codex, try using LiteLLM’s `use_chat_completions_api`."
description: "This article explains how to use the `use_chat_completions_api: true` setting in LiteLLM to convert Codex CLI's Responses API requests to the Chat Completions API, thereby avoiding errors related to tool calls and image input."
pubDatetime: 2026-11-25T15:26:00+09:00
---
If you connect Codex to an OpenAI-compatible model server, you may encounter errors when trying to call MCP tools or attach screenshots, even though simple text interactions work. The Codex CLI uses the Responses API to communicate with the model, but some backends may only accept tool calls and image inputs through the Chat Completions API. [LiteLLM Codex Connection Guide](https://docs.litellm.ai/docs/proxy/client_setup/codex_cli) · [OpenAI Image Input Guide](https://developers.openai.com/api/docs/guides/images-vision)

In such cases, try setting `use_chat_completions_api: true` in LiteLLM. Codex sends requests to `/v1/responses`, and LiteLLM converts them to `/v1/chat/completions` to match the backend. This setting can also be used if the backend does not support the Responses API and even simple text interactions result in a 404 error. [LiteLLM Responses API Guide](https://docs.litellm.ai/docs/response_api#opt-in-bridge-for-openai-models-with-custom-api_base)

## Test Failing Inputs One by One

First, send only text, then try attaching an image, and finally, try calling one MCP tool. Examine the logs of LiteLLM and the backend. If the model API is rejecting the request, check if the `/v1/chat/completions` endpoint for the same model can handle the corresponding image input or function call. If it works there, then it may be worth trying the conversion. If the connection or authentication to the MCP server fails, check the server's configuration. [LiteLLM Image Input Guide](https://docs.litellm.ai/docs/completion/vision) · [Codex MCP Configuration Guide](https://developers.openai.com/codex/mcp)

## Configure in LiteLLM Proxy for the Target Model

Add `use_chat_completions_api: true` to the `litellm_params` of the target model in `config.yaml`. LiteLLM receives the request from the client at `/v1/responses`, sends it to the backend at `/v1/chat/completions`, and converts the result back to the Responses API format. [LiteLLM Official Documentation](https://docs.litellm.ai/docs/response_api)

```yaml
model_list:
  - model_name: local-chat
    litellm_params:
      model: openai/my-served-model
      api_base: http://127.0.0.1:8080/v1
      api_key: os.environ/UPSTREAM_API_KEY
      use_chat_completions_api: true

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

Replace `my-served-model` with the model ID exposed by the backend. The name used to specify the model from the client is `local-chat`. In this example, the backend key is set to `UPSTREAM_API_KEY`, and the proxy master key is set to `LITELLM_MASTER_KEY` before starting the service. [LiteLLM YAML Configuration Guide](https://docs.litellm.ai/docs/proxy/configs)

```bash
litellm --config config.yaml --port 4000
```

When using from the Codex CLI, specify `local-chat` in the model name in `~/.codex/config.toml`. For the LiteLLM provider settings, use `base_url = "http://127.0.0.1:4000/v1"`, `env_key = "LITELLM_API_KEY"`, and `wire_api = "responses"`. [LiteLLM Codex Connection Guide](https://docs.litellm.ai/docs/proxy/client_setup/codex_cli)

In this local verification, call it with the same value as `LITELLM_MASTER_KEY` set to `LITELLM_API_KEY` in another terminal.

```bash
curl -sS http://127.0.0.1:4000/v1/responses \
  -H "Authorization: Bearer $LITELLM_API_KEY" \
  -H 'Content-Type: application/json' \
  --data '{"model":"local-chat","input":"Please give a short greeting in English."}'
```

If the `output` in the response contains text and `POST /v1/chat/completions` is logged by the backend, the conversion is working.

## Pass `True` in the Python SDK

When connecting from the LiteLLM Python SDK without going through a proxy, you can pass the same setting as a keyword argument to `litellm.responses()`. [LiteLLM Official Documentation](https://docs.litellm.ai/docs/response_api)

```python
import os

import litellm

response = litellm.responses(
    model="openai/my-served-model",
    api_base="http://127.0.0.1:8080/v1",
    api_key=os.environ["UPSTREAM_API_KEY"],
    input="日本語で短く挨拶してください。",
    use_chat_completions_api=True,
)
print(response)
```

## After Switching, Try the Inputs That Previously Failed Again

If you get a short text response, try attaching the original image again in Codex and check if you get a response related to the image content. For MCP, try until the tool is actually called and the response can be generated using its result. Image input requires the backend model and API to support images. The operation of MCP tools also depends on the connection settings on the Codex side and the support for the tool format passed to the model. [LiteLLM Image Input Guide](https://docs.litellm.ai/docs/completion/vision) · [LiteLLM Report on Tool Conversion](https://github.com/BerriAI/litellm/issues/35878)
