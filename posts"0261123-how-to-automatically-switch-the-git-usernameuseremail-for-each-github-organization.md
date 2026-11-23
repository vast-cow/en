---
title: "How to automatically switch Git user.name / user.email for each GitHub organization."
description: "This article explains how to automatically switch between different user.name and user.email configurations based on the remote URL using Git’s includeIf feature."
pubDatetime: 2026-11-23T10:48:00+09:00
---
If you use multiple GitHub accounts or organizations, it can be a hassle to set the Git `user.name` and `user.email` for each repository.

For example, consider the following cases:

- You want to use your personal email address for your personal repositories.
- You want to use your work email address for your company's organization.
- You want to use a different email address for a specific open-source repository.

Normally, you would configure these settings for each repository using `git config --local`. However, by using Git's **`includeIf`**, you can automatically switch Git settings based on the remote repository URL.

Moreover, you don't need to prepare Git hooks or external tools.

This article explains how to use `includeIf "hasconfig:remote.*.url"` to achieve Git identity switching based on the GitHub repository owner.

## 1. What is `includeIf`?

`includeIf` is a feature that loads another Git configuration file if a specified condition is met.

The typical conditions are as follows:

| Condition | Use case |
|---|---|
| `gitdir:` | Checks the repository directory |
| `onbranch:` | Checks the current branch |
| `hasconfig:remote.*.url:` | Checks the remote URL |

We will be using `hasconfig:remote.*.url` in this case.

It is available in Git 2.36 and later and allows you to load configuration files based on the remote URL configured in the repository.

For example, you can configure it as follows:

```gitconfig
[includeIf "hasconfig:remote.*.url:git@github.com:my-company/**"]
    path = ~/.config/git/identities/work.gitconfig
```

This means that if a remote URL that matches `git@github.com:my-company/` or its subdirectories exists, `work.gitconfig` will be loaded.

The important point is that the condition is based on the **GitHub remote URL, not the directory**.

Regardless of which directory the repository is cloned into, the same settings will be applied.

## 2. Divide the configuration files

First, prepare configuration files for each identity.

Example structure:

```text
~
├── .gitconfig
└── .config/
    └── git/
        └── identities/
            ├── personal.gitconfig
            ├── work.gitconfig
            └── oss.gitconfig
```

Create the directory:

```bash
mkdir -p ~/.config/git/identities
```

### Personal identity

`~/.config/git/identities/personal.gitconfig`

```gitconfig
[user]
    name = My Name
    email = personal@example.com
```

### Work identity

`~/.config/git/identities/work.gitconfig`

```gitconfig
[user]
    name = My Name
    email = me@company.com
```

### Open-source identity

`~/.config/git/identities/oss.gitconfig`

```gitconfig
[user]
    name = My Name
    email = oss@example.com
```

Each file contains `user.name` and `user.email` in the same format as a normal Git configuration.

## 3. Switch based on remote URL

Next, configure conditional loading in `~/.gitconfig`.

```gitconfig
[user]
    useConfigOnly = true

# Personal account
[includeIf "hasconfig:remote.*.url:git@github.com:my-personal/**"]
    path = ~/.config/git/identities/personal.gitconfig

[includeIf "hasconfig:remote.*.url:https://github.com/my-personal/**"]
    path = ~/.config/git/identities/personal.gitconfig

# Company organization
[includeIf "hasconfig:remote.*.url:git@github.com:my-company/**"]
    path = ~/.config/git/identities/work.gitconfig

[includeIf "hasconfig:remote.*.url:https://github.com/my-company/**"]
    path = ~/.config/git/identities/work.gitconfig
```

To support both SSH and HTTPS, we configure both URL formats.

This allows for the following switching:

| Remote URL | Applied identity |
|---|---|
| `git@github.com:my-personal/app.git` | personal |
| `https://github.com/my-personal/app.git` | personal |
| `git@github.com:my-company/backend.git` | work |
| `https://github.com/my-company/frontend.git` | work |

**You do not need to execute `git config --local`.**

Each time Git loads the configuration, it checks the remote URLs configured in the current repository and selects the appropriate identity.

### About `user.useConfigOnly`

The configuration starts with the following item:

```gitconfig
[user]
    useConfigOnly = true
```

This setting prevents Git from inferring the commit identity from the OS user name.

If you do not set a default `user.name` / `user.email` globally, a commit will fail if there is no valid identity when no `includeIf` matches.

This is useful for preventing accidental commits with the wrong email address, such as committing to a personal repository with your work email address.

## 4. Set exceptions for specific repositories

In addition to organization units, you can also apply different identities to specific repositories.

For example, you want to use the work identity for repositories under `my-company` in general, but use the open-source identity for `my-company/opensource`.

Add the following to the end of `~/.gitconfig`:

```gitconfig
# Exception for specific repositories
[includeIf "hasconfig:remote.*.url:git@github.com:my-company/opensource.git"]
    path = ~/.config/git/identities/oss.gitconfig

[includeIf "hasconfig:remote.*.url:https://github.com/my-company/opensource.git"]
    path = ~/.config/git/identities/oss.gitconfig
```

Since Git configuration values loaded later take precedence, you can overwrite the organization's settings by writing the repository-specific settings after them.

The result will be:

```text
my-company/
├── backend       → work
├── frontend      → work
└── opensource    → oss
```

Note that in this example, we are using an exact match for URLs ending in `.git`. If your actual remote URLs do not have `.git` at the end, you will need to add a condition for that format as well.

## 5. Check if the settings are reflected

Check the remote URL in a suitable repository.

```bash
git remote -v
```

Example:

```text
origin  git@github.com:my-company/backend.git (fetch)
origin  git@github.com:my-company/backend.git (push)
```

Next, check the effective identity.

```bash
git config --get user.name
git config --get user.email
```

Result:

```text
My Name
me@company.com
```

You can also check which configuration file is being loaded.

```bash
git config --show-origin --get user.email
```

Result:

```text
file:/home/user/.config/git/identities/work.gitconfig  me@company.com
```

The following is also useful if there are multiple settings.

```bash
git config --show-origin --get-all user.email
```

This allows you to investigate which file is setting the value.

## 6. Precautions

### It does not only judge `origin`

`hasconfig:remote.*.url` is not limited to `origin`.

**Any remote URL that matches the condition will trigger the condition.**

For example, consider the following repository:

```text
origin   git@github.com:my-personal/fork.git
upstream git@github.com:my-company/original.git
```

In this case, both the personal and work conditions may be met.

Therefore, caution is required when dealing with repositories that belong to multiple organizations.

If you want to strictly judge only `origin`, you cannot do so with `includeIf` alone, and you will need custom processing using Git hooks or the like.

### `git config --local` takes precedence

If a local configuration exists for the repository, that configuration usually takes precedence.

```bash
git config --local user.email override@example.com
```

If this setting exists, the local setting takes precedence over the global scope email address loaded by `includeIf`.

The same applies if an existing hook was writing to `git config --local`.

If you are completely transitioning to automatic switching, check the local settings of the relevant repository and remove any unnecessary settings.

```bash
git config --local --unset user.name
git config --local --unset user.email
```

However, do not delete intentionally set local identities.

### Do not include remote URL in the configuration file

In the configuration files loaded by `hasconfig:remote.*.url`, you cannot define `remote.*.url`.

For example, you need to avoid a configuration that sets the remote URL in `work.gitconfig`.

This restriction is to prevent circular dependencies where the loaded configuration file itself affects the condition check.

## 7. Advantages compared to Git hook

Previously, I was considering using a `post-checkout` hook to retrieve the remote URL and obtain user information from the GitHub API and save it to `git config --local`.

However, if the identity is fixed, `includeIf` is simpler.

| Comparison item | post-checkout hook | includeIf |
|---|---|---|
| External script | Required | Not required |
| GitHub API | Available | Not required |
| Switching based on remote URL | Possible | Possible |
| Configuration by organization unit | Possible | Possible |
| Exceptions by repository unit | Possible | Possible |
| Judgment limited to `origin` | Possible | Not possible |
| Writing to local configuration | Depends on implementation | Not required |
| Reflecting configuration changes | May require re-executing the hook | Reflected during the next configuration loading |

The main difference is that `includeIf` **does not permanently modify the configuration**.

For example, if you change the remote URL of a repository, the identity will be switched based on the new condition.

Also, if you change the email address in `work.gitconfig`, you do not need to update the local settings of each repository.

## Summary

If you want to switch commit identities based on GitHub organization or repository, Git's `includeIf "hasconfig:remote.*.url"` is convenient.

- Can be implemented using only Git's standard features
- Supports both SSH and HTTPS
- Can be switched by organization unit and repository unit
- No need to write to `git config --local`
- Configuration file changes can be reflected in existing repositories

However, be careful if multiple remote URLs match the condition, or if you want to strictly judge only `origin`.

**If the correspondence between identities is determined statically, use `includeIf`. If dynamic judgment using the GitHub API or complex condition branching is required, use a hook.**

### References

- [Git Official Documentation: git-config](https://git-scm.com/docs/git-config)
- [Git 2.36 Release Notes](https://github.com/git/git/blob/master/Documentation/RelNotes/2.36.0.adoc)
