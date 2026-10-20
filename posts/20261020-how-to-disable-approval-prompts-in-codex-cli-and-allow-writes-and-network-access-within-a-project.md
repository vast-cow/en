---
title: "How to Disable Approval Prompts in Codex CLI and Allow Writes and Network Access Within a Project"
description: "Three settings in the Codex CLI configuration file enable autonomous file writing, command execution without approval, and network access within the sandbox."
pubDatetime: 2026-10-20T13:15:00+09:00
---

When using the OpenAI Codex CLI, you may be prompted for approval every time you edit a file or execute a command.

While this is an important security measure, it can be cumbersome if you want Codex to work more autonomously on development projects.

This article describes how to configure the following three settings in the Codex CLI:

- Allow file writing within the project
- Disable approval prompts when executing commands
- Allow network access from within the sandbox

The configuration is done using `~/.codex/config.toml`.

## 1. Edit the Codex CLI configuration file

Codex CLI settings are generally managed in `~/.codex/config.toml`.

On macOS or Linux, run the following in the terminal:

```bash
mkdir -p ~/.codex
nano ~/.codex/config.toml
```

If the configuration file already exists, edit it while preserving the existing settings.

## 2. Set `sandbox_mode` and `approval_policy`

Add the following to the configuration file:

```toml
sandbox_mode = "workspace-write"
approval_policy = "never"

[sandbox_workspace_write]
network_access = true
```

This configuration will allow writing within the workspace, disable approval prompts, and allow network access within the sandbox.

Let's examine each setting.

### `sandbox_mode = "workspace-write"`

```toml
sandbox_mode = "workspace-write"
```

This setting specifies the scope of the file system that Codex can access.

`workspace-write` allows writing to the workspace.

Note that the value is `workspace-write`, not `project-write`.

### `approval_policy = "never"`

```toml
approval_policy = "never"
```

This setting specifies the approval policy when Codex executes commands.

Setting it to `never` will allow Codex to proceed without prompting the user for approval.

However, disabling approval prompts is different from allowing all operations.

Operations prohibited by the sandbox are not automatically allowed.

### `network_access = true`

```toml
[sandbox_workspace_write]
network_access = true
```

This allows network access within the `workspace-write` sandbox.

This setting enables commands that require network access, such as making HTTP requests to external APIs or retrieving dependent packages from the internet.

The value to specify is `true`, a boolean, not the string `"enabled"`.

Note that the availability of network access is also affected by the execution environment and higher-level security restrictions.

## 3. How to specify temporarily from the command line

You can also specify the settings using command-line options without modifying the configuration file.

```bash
codex \
  --sandbox workspace-write \
  --ask-for-approval never \
  -c sandbox_workspace_write.network_access=true
```

This method is useful when you want to enable autonomous execution only for specific tasks, while keeping approval prompts enabled most of the time.

You can use the method of writing to the configuration file and the method of specifying options at startup according to the use case.

## 4. Important considerations when configuring

With the settings in this article, Codex will be able to modify files and access the network without waiting for user confirmation.

Therefore, be careful about the following:

- Commit the state before working with Git.
- Do not handle sensitive information such as `.env` files carelessly.
- Do not use it carelessly in untrusted repositories.
- Do not run it with production environment credentials.
- Adjust the sandbox and approval policy as needed.

In particular, specifying `approval_policy = "never"` reduces the opportunity to notice dangerous operations with approval prompts.

Configure the settings considering the balance between automation convenience and security.

## Summary

If you want to enable file writing within the project, run without approval prompts, and allow network access in the Codex CLI, use the following settings:

```toml
sandbox_mode = "workspace-write"
approval_policy = "never"

[sandbox_workspace_write]
network_access = true
```

The key points of the configuration are as follows:

1. Specify `workspace-write` for `sandbox_mode`.
2. Specify `never` for `approval_policy`.
3. Specify `network_access = true` for network access.

This will allow the Codex CLI to operate more autonomously.

However, even if you disable approval prompts, the sandbox restrictions remain, so operations outside the workspace will not be fully allowed.

Use this in local development or verification environments to improve the efficiency of Codex CLI operations.

Reference: [OpenAI Official Documentation: Codex Configuration Reference](https://developers.openai.com/ja-JP/docs/config-file/config-reference). The values for `workspace-write`, `never`, and `network_access` have been verified in the official specifications.
