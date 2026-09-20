---
title: "Using Multiple GitHub Accounts: Automatically Select the Correct `gh` PAT Based on Repo Owner"
description: "When using multiple GitHub accounts, the process of selecting the correct Personal Access Token (PAT)..."
pubDatetime: 2026-08-26T07:16:37.440Z
---

When using multiple GitHub accounts, the process of selecting the correct Personal Access Token (PAT) for `git push` or `git pull` over HTTPS can be a bit cumbersome.

Specifically, when dealing with repositories from multiple owners, such as:

```text
https://github.com/aont/foo.git
https://github.com/another-user/bar.git
```

Git Credential Manager or `gh auth git-credential` might default to the currently active account, leading to errors like:

```text
remote: Permission to owner/repo.git denied to other-user.
```

To address this, the `owner` portion of the remote URL is interpreted as the GitHub CLI account name, and a credential helper is embedded directly into the `.gitconfig` file. This helper retrieves the PAT using:

```bash
gh auth token --user OWNER
```

and provides it to Git.

This eliminates the need for external scripts.

## Prerequisites

First, ensure that you are logged into your GitHub accounts using `gh`:

```bash
gh auth login
```

If you have multiple accounts registered, you can verify them with:

```bash
gh auth status
```

With this method, if a repository URL is:

```text
https://github.com/aont/foo.git
```

the command:

```bash
gh auth token --user aont
```

will be executed.

Therefore, the basic assumption is:

```text
repository owner = gh registered account name
```

## `.gitconfig`

The configuration should be as follows:

```gitconfig
[credential]
    useHttpPath = true

[credential "https://github.com/"]
    helper =
    helper = "!f() { \
        [ \"$1\" = get ] || exit 0; \
        account=''; \
        while IFS='=' read -r k v; do \
            [ \"$k\" = path ] && account=${v%%/*}; \
        done; \
        [ -n \"$account\" ] || exit 0; \
        token=$(gh auth token --user \"$account\") || exit 0; \
        printf 'username=%s\\npassword=%s\\n' \"$account\" \"$token\"; \
    }; f"

[credential "https://gist.github.com/"]
    helper =
    helper = "!f() { \
        [ \"$1\" = get ] || exit 0; \
        account=''; \
        while IFS='=' read -r k v; do \
            [ \"$k\" = path ] && account=${v%%/*}; \
        done; \
        [ -n \"$account\" ] || exit 0; \
        token=$(gh auth token --user \"$account\") || exit 0; \
        printf 'username=%s\\npassword=%s\\n' \"$account\" \"$token\"; \
    }; f"
```

There are three key points:

## `credential.useHttpPath = true` is Important

Typically, Git's credential helper receives only:

```text
protocol=https
host=github.com
```

However, in this case, we need:

```text
aont/foo.git
```

the path.

Therefore, set:

```gitconfig
[credential]
    useHttpPath = true
```

This causes the path to be passed to the credential helper's standard input, such as:

```text
protocol=https
host=github.com
path=aont/foo.git
```

## Use the Beginning of the Path as the Account

Inside the helper, the first element of the path is extracted using:

```sh
[ "$k" = path ] && account=${v%%/*}
```

For example, if:

```text
path=aont/foo.git
```

then:

```text
account=aont
```

After that, the corresponding PAT for that account is retrieved from the GitHub CLI using:

```sh
token=$(gh auth token --user "$account")
```

Therefore, when accessing:

```text
https://github.com/aont/foo.git
```

the command:

```bash
gh auth token --user aont
```

is automatically executed.

Conversely, for:

```text
https://github.com/another-user/bar.git
```

the command:

```bash
gh auth token --user another-user
```

is executed.

There is no need to run `gh auth switch` every time.

## Reset Existing Credential Helpers with `helper =`

Another important point is:

```gitconfig
helper =
```

On Git for Windows, for example, Git Credential Manager or `gh auth git-credential` might already be configured.

If another helper returns the credentials first, this helper might not be executed.

As a result, you might encounter an error like:

```text
remote: Permission to foo/bar.git denied to wrong-account.
```

Therefore, use:

```gitconfig
helper =
helper = "!f() { ... }; f"
```

The first empty `helper =` resets any previously configured credential helpers, and then only this helper is registered.

This is crucial when dealing with multiple GitHub accounts.

## Use the Same Mechanism for Gists

The same credential helper can be configured for Gists.

However, the standard Gist clone URL is:

```text
https://gist.github.com/GIST_ID.git
```

and the username cannot be determined from the URL.

Therefore, in this setup, the Gist remote URL is intentionally formatted as:

```text
https://gist.github.com/USER/GIST_ID.git
```

For example, if:

```text
https://gist.github.com/aont/0123456789abcdef.git
```

then, from the credential helper's perspective:

```text
path=aont/0123456789abcdef.git
```

so, the command:

```bash
gh auth token --user aont
```

is automatically used.

This allows you to treat GitHub repositories and Gists with the same rules.

```text
github.com/USER/REPO
gist.github.com/USER/GIST
                ↓
             USER is extracted
                ↓
gh auth token --user USER
```

## Verification

You can check which credentials Git is actually retrieving using `git credential fill`.

For example, run:

```bash
printf '%s\n' \
  'protocol=https' \
  'host=github.com' \
  'path=aont/foo.git' \
  '' |
git credential fill
```

The expected output is:

```text
protocol=https
host=github.com
username=aont
password=...
```

If the `username` is a different account, you can check if another credential helper is still present using:

```bash
git config --show-origin --get-all credential.helper
```

## Why Not Use `gh auth git-credential` Directly?

The GitHub CLI has a standard command:

```bash
gh auth git-credential
```

This is sufficient for normal single-account usage.

However, when using multiple accounts with the same host (`github.com`), you might want to explicitly control which `gh` account is used based on the repository owner.

This helper uses a very simple rule:

```text
remote URL
  ↓
owner is extracted
  ↓
gh auth token --user owner
```

so you don't need to be aware of which account is currently active in `gh`.

## Advantages of This Configuration

This method does not save the PAT itself in the `.gitconfig` file.

The PAT is retrieved each time it is needed using:

```bash
gh auth token --user ACCOUNT
```

Therefore, the configuration only saves the rule for which account to use.

Additionally, you do not need to run:

```bash
gh auth switch
```

or set:

```bash
git config credential.username ...
```

for each repository.

The remote URL itself becomes the credential routing information.

## Summary

When using multiple GitHub accounts over HTTPS, using the `account` portion of:

```text
github.com/<account>/<repo>
```

directly for selecting the `gh` account can simplify operations.

The mechanism is:

```text
Git remote URL
  ↓
credential.useHttpPath
  ↓
path=owner/repo.git
  ↓
owner is extracted
  ↓
gh auth token --user owner
  ↓
username/password is returned to Git
```

If you have multiple personal accounts and the relationship:

```text
repository owner = GitHub account
```

holds true, this method is quite easy to use.

For organization-owned repositories where:

```text
owner != account used for authentication
```

you will need separate mappings, but this configuration is sufficient for primarily personal accounts.
