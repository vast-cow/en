---
title: "Notes on Tabby: Llama.cpp, Model Caching, and Access Tokens"
description: "Tabby is a developer-focused tool that can run and manage local AI models, and it includes a few..."
pubDatetime: 2026-01-20T02:51:21.500Z
---

Tabby is a developer-focused tool that can run and manage local AI models, and it includes a few practical configuration and account details that are useful to keep in mind.

## Tabby uses `llama.cpp` internally

One notable point is that Tabby uses `llama.cpp` under the hood. In practice, this means Tabby can leverage the lightweight, local-inference approach that `llama.cpp` is known for, which is often used to run LLMs efficiently on local machines.

## Model cache location: `TABBY_MODEL_CACHE_ROOT`

Tabby supports configuring where it stores cached model files. The environment variable:

* `TABBY_MODEL_CACHE_ROOT`

is used to control the root directory for Tabby’s model cache. If you want to manage disk usage, place models on a faster drive, or standardize storage paths across multiple environments, setting this variable is a straightforward way to do it.

## Registry reference: `registry-tabby`

Tabby’s model registry can be found here:

* [https://github.com/TabbyML/registry-tabby](https://github.com/TabbyML/registry-tabby)

This repository is a useful reference point for understanding what models are available or how Tabby’s registry entries are organized.

## How to check your token

To view the token you need for authenticated access, you typically:

1. Access the service via a web browser.
2. Create an account and log in.
3. After logging in, you can locate and confirm your token from the account or user settings area.

In other words, the token is something you can verify after a one-time browser-based account setup and login.
