---
title: "How to Cache LLM Outputs with LiteLLM"
description: "Disk caching in LiteLLM stores LLM responses, reducing API costs and latency for repeated requests."
pubDatetime: 2026-09-29T23:14:00+09:00
---

When developing with LLMs, running the same prompt repeatedly incurs inference time and API costs each time.

LiteLLM provides a caching function that allows you to save and reuse LLM outputs for identical requests. This section introduces the simplest configuration using disk caching.

First, install the dependencies required for caching and the proxy.

```bash
pip install 'litellm[caching,proxy]'
```

Next, create the LiteLLM Proxy configuration file.

```yaml
model_list:
  - model_name: model
    litellm_params:
      model: openai/Qwen/Qwen3.6-35B-A3B # example
      api_key: ...
      api_base: ...

litellm_settings:
  cache: true
  cache_params:
    type: disk
    disk_cache_dir: ./Qwen_Qwen3.6-35B-A3B
```

Note that `openai/Qwen/Qwen3.6-35B-A3B` starts with `openai/`, which is necessary to indicate that it uses the OpenAI-compatible API.

The key is the `litellm_settings` configuration.

Setting `cache: true` enables caching, and specifying `cache_params.type` as `disk` allows you to save LLM responses to the local disk.

```yaml
cache_params:
  type: disk
  disk_cache_dir: ./Qwen_Qwen3.6-35B-A3B
```

In this example, the cached data is saved in the `./Qwen_Qwen3.6-35B-A3B` directory.

If the same model and the same request are sent again, the cache can be used, which eliminates the need to query the LLM, leading to shorter response times and reduced API costs.

This is particularly useful for applications that repeatedly execute the same input, such as evaluation scripts, benchmarks, and repetitive testing during development.

If you are using LiteLLM, try disk caching first as it is easy to set up.
