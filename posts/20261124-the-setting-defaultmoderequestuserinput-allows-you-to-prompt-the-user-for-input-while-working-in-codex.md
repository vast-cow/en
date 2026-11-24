---
title: "How to configure Codex to ask the user questions while working: “default_mode_request_user_input”"
description: "A feature flag that enables user input requests in Codex’s Default mode. By adding just two lines to the config.toml file, you can use questions with multiple-choice options."
pubDatetime: 2026-11-24T12:08:00+09:00
---
Sometimes, when I let Codex handle the development, I feel like it should have checked with me instead of making decisions on its own.

For example, this happens when there are multiple ways to implement something or when significant changes to existing code are required.

This is where the `default_mode_request_user_input` feature flag in Codex comes in handy.

By simply adding two lines to `config.toml`, you can enable Codex to ask users questions with selectable options even in the normal Default mode.

## What is `default_mode_request_user_input`?

Codex has a tool called `request_user_input` to obtain additional information from the user.

This tool allows Codex to present questions to the user during a task and incorporate the answers into the process.

Originally, this feature was available in Plan mode, but it was experimentally added to Default mode in Codex CLI v0.106.0.

The key feature is that it allows you to create interactive workflows that go beyond simply asking questions in a normal chat, such as presenting choices.

For example, it can be used in the following situations:

- When there are multiple implementation methods, let the user choose.
- When the specifications are ambiguous, confirm additional information.
- Before changing the file structure, confirm the user's preferences.
- In design and requirements definition, ask questions in a step-by-step manner.

## How to set it up

Setting it up is very easy.

### 1. Open `config.toml`

Edit the Codex configuration file.

It is usually located at:

```text
~/.codex/config.toml
```

If the file doesn't exist, create it.

### 2. Add features

Add the following to the configuration file:

```toml
[features]
default_mode_request_user_input = true
```

If a `[features]` section already exists, simply add `default_mode_request_user_input = true` to it.

The setup is now complete. Restart Codex to apply the changes.

### 3. Verify that the setting is enabled

Run the following command in the terminal:

```bash
codex features list
```

If `default_mode_request_user_input` is enabled, the setting is correct.

You can also temporarily enable it without editing the configuration file:

```bash
codex --enable default_mode_request_user_input
```

## Let's try it out

For example, instruct Codex as follows:

> Improve the README.
>
> However, before starting, use `request_user_input` to ask what the target audience is: "Beginner" or "Experienced."
>
> Adjust the README content based on the answer.

In a compatible environment, Codex will display a question with selectable options.

When the user answers, the task will proceed based on that information.

It's important to remember that not only can you let Codex decide when to ask questions, but you can also **explicitly instruct it to ask questions from the prompt**.

This expands the interaction with Codex from a simple task request to a development workflow that involves dialogue.

## Important Note: It can't wait indefinitely

This is an important point to keep in mind.

`default_mode_request_user_input = true` configures the use of the question tool in Default mode and does not necessarily configure Codex to stop and wait until the user answers.

In Default mode, if the user does not answer within a certain time, the question is automatically resolved.

Therefore, be cautious when using it for the following purposes:

- Don't change the production environment until the user approves.
- Don't stop the task for several hours until the user responds.
- Don't resume processing until explicit permission is granted.

For such strict approval control, you need to prepare a separate approval flow or mechanism to manage the waiting state, in addition to relying solely on the question function.

Also, `request_user_input` and Codex's command execution approval policy are separate mechanisms.

## Is MCP necessary?

If you only use the feature introduced this time, you don't need to add an MCP server.

`request_user_input` is a function within Codex.

On the other hand, if your own MCP tool requires user input, it uses a separate mechanism called Elicitation in MCP.

In other words, if Codex itself is checking specifications, use `request_user_input`, and if a custom tool is requesting user input, use MCP Elicitation.

## Summary

By adding the following setting to Codex, you can use the user input request function even in the normal Default mode.

```toml
[features]
default_mode_request_user_input = true
```

In particular, it is a useful feature for development tasks that require user judgment, such as requirements definition, selection of design approaches, and modification of existing code.

However, it is currently in development, so its behavior may vary depending on the version of Codex and the environment in which it is used.

Personally, I think this is a useful setting for using Codex not just as an AI that writes code, but as an **agent that can be used to advance development by asking questions when necessary**.

## References

- [Codex Configuration Reference](https://developers.openai.com/codex/config-reference)
- [Codex feature flag implementation (GitHub)](https://github.com/openai/codex/blob/main/codex-rs/features/src/lib.rs)
- [Support for `request_user_input` in Default mode](https://github.com/openai/codex/issues/10384)
- [Discussion about indefinite waiting in Default mode](https://github.com/openai/codex/issues/37472)

*Based on information as of November 2026.*
