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

To address this, the `owner` portion of the remote URL can be used to determine the GitHub CLI account, and a credential helper can be embedded directly into the `.gitconfig` file. This helper retrieves the PAT using:

```bash
gh auth token --user ACCOUNT
```

and provides it to Git.

For personal repositories, the repository owner can be used directly as the account name. For organization-owned repositories, where the repository owner and the account used for authentication differ, explicit mappings can be added.

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

With the basic method, if a repository URL is:

```text
https://github.com/aont/foo.git
```

the command:

```bash
gh auth token --user aont
```

will be executed.

Therefore, the default assumption is:

```text
repository owner = gh registered account name
```

When this assumption does not hold, explicit mappings can override it.

## `.gitconfig`

A configuration supporting both the default owner-based behavior and explicit account mappings can be written as follows:

```gitconfig
[credential]
    useHttpPath = true

[credential "https://github.com/"]
    helper =
    helper = "!f() { \
        [ \"$1\" = get ] || exit 0; \
        path=''; \
        while IFS='=' read -r k v; do \
            [ \"$k\" = path ] && path=$v; \
        done; \
        [ -n \"$path\" ] || exit 0; \
        owner=${path%%/*}; \
        repo=${path#*/}; \
        repo=${repo%.git}; \
        case \"$owner/$repo\" in \
            my-org/special-repo) account='special-account' ;; \
            *) \
                case \"$owner\" in \
                    my-org) account='my-work-account' ;; \
                    another-org) account='another-account' ;; \
                    *) account=\"$owner\" ;; \
                esac \
                ;; \
        esac; \
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

There are four key points.

## `credential.useHttpPath = true` is Important

Typically, Git's credential helper receives only:

```text
protocol=https
host=github.com
```

However, in this case, we need the path:

```text
aont/foo.git
```

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

## Use the Beginning of the Path as the Default Account

Inside the helper, the path is first stored:

```sh
path=''
while IFS='=' read -r k v; do
    [ "$k" = path ] && path=$v
done
```

The repository owner is then extracted using:

```sh
owner=${path%%/*}
```

and the repository name is extracted using:

```sh
repo=${path#*/}
repo=${repo%.git}
```

For example, if:

```text
path=aont/foo.git
```

then:

```text
owner=aont
repo=foo
```

Unless an explicit mapping exists, the owner is used directly as the GitHub CLI account:

```sh
account="$owner"
```

The corresponding PAT is then retrieved using:

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

Similarly, for:

```text
https://github.com/another-user/bar.git
```

the command:

```bash
gh auth token --user another-user
```

is executed.

There is no need to run `gh auth switch` every time.

## Map Organization Owners to GitHub Accounts

The simple `owner = account` rule works well for repositories owned by personal accounts, but it does not necessarily work for organization-owned repositories.

For example, suppose the remote URL is:

```text
https://github.com/my-org/foo.git
```

but the GitHub account that has access to the organization is:

```text
my-work-account
```

In that case:

```text
repository owner = my-org
authentication account = my-work-account
```

so executing:

```bash
gh auth token --user my-org
```

would be incorrect.

The helper can handle this with an owner-level mapping:

```sh
case "$owner" in
    my-org) account='my-work-account' ;;
    another-org) account='another-account' ;;
    *) account="$owner" ;;
esac
```

This means:

```text
github.com/my-org/foo
        ↓
owner = my-org
        ↓
account = my-work-account
        ↓
gh auth token --user my-work-account
```

Any owner that is not explicitly listed continues to use the original behavior:

```text
account = owner
```

so personal repositories do not require any additional configuration.

## Override the Account for a Specific Repository

Sometimes repositories under the same organization need to use different accounts.

For example:

```text
https://github.com/my-org/foo.git
https://github.com/my-org/special-repo.git
```

might normally use:

```text
my-work-account
```

but `special-repo` might need:

```text
special-account
```

The helper handles this by checking the full `owner/repo` combination before checking the owner:

```sh
case "$owner/$repo" in
    my-org/special-repo) account='special-account' ;;
    *)
        case "$owner" in
            my-org) account='my-work-account' ;;
            another-org) account='another-account' ;;
            *) account="$owner" ;;
        esac
        ;;
esac
```

The precedence is therefore:

```text
repository-specific mapping
        ↓
organization/owner mapping
        ↓
owner name as the default account
```

For example:

```text
github.com/my-org/special-repo
        ↓
special-account

github.com/my-org/other-repo
        ↓
my-work-account

github.com/aont/foo
        ↓
aont
```

This makes it possible to handle personal repositories, organization-owned repositories, and repository-specific exceptions with the same credential helper.

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

so the command:

```bash
gh auth token --user aont
```

is automatically used.

This allows GitHub repositories and Gists to follow a similar rule:

```text
github.com/OWNER/REPO
gist.github.com/USER/GIST
                ↓
       account is determined
                ↓
gh auth token --user ACCOUNT
```

For normal personal repositories and Gists, the owner or user is used directly. For organization-owned repositories, explicit mappings can override that behavior.

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

You can also verify an organization mapping:

```bash
printf '%s\n' \
  'protocol=https' \
  'host=github.com' \
  'path=my-org/foo.git' \
  '' |
git credential fill
```

With the configuration above, the expected username is:

```text
username=my-work-account
```

For the repository-specific override:

```bash
printf '%s\n' \
  'protocol=https' \
  'host=github.com' \
  'path=my-org/special-repo.git' \
  '' |
git credential fill
```

the expected username is:

```text
username=special-account
```

If the `username` is unexpected, you can check whether another credential helper is still present using:

```bash
git config --show-origin --get-all credential.helper
```

## Why Not Use `gh auth git-credential` Directly?

The GitHub CLI has a standard command:

```bash
gh auth git-credential
```

This is sufficient for normal single-account usage.

However, when using multiple accounts with the same host (`github.com`), you might want to explicitly control which `gh` account is used based on the repository owner or repository itself.

This helper uses the following routing rule:

```text
remote URL
  ↓
owner/repository is extracted
  ↓
repository-specific mapping, if any
  ↓
owner mapping, if any
  ↓
otherwise use owner as account
  ↓
gh auth token --user account
```

As a result, you do not need to be aware of which account is currently active in `gh`.

## Advantages of This Configuration

This method does not save the PAT itself in the `.gitconfig` file.

The PAT is retrieved each time it is needed using:

```bash
gh auth token --user ACCOUNT
```

Therefore, the configuration only saves the rule for determining which account to use.

Additionally, you do not need to run:

```bash
gh auth switch
```

or set:

```bash
git config credential.username ...
```

for each repository.

The remote URL itself becomes the input for credential routing.

For most personal repositories, no explicit configuration is necessary:

```text
github.com/aont/foo
        ↓
account = aont
```

Organization accounts can be mapped:

```text
github.com/my-org/foo
        ↓
account = my-work-account
```

and individual repositories can override that mapping:

```text
github.com/my-org/special-repo
        ↓
account = special-account
```

## Summary

When using multiple GitHub accounts over HTTPS, the repository path can be used to automatically select the appropriate `gh` account.

The mechanism is:

```text
Git remote URL
  ↓
credential.useHttpPath
  ↓
path=owner/repo.git
  ↓
owner and repo are extracted
  ↓
repository-specific mapping?
  ├─ yes → use mapped account
  └─ no
       ↓
     owner mapping?
       ├─ yes → use mapped account
       └─ no → use owner as account
  ↓
gh auth token --user account
  ↓
username/password is returned to Git
```

The default rule remains simple:

```text
repository owner = GitHub account
```

but explicit mappings make the same approach usable for organization-owned repositories where:

```text
repository owner != account used for authentication
```

Repository-specific overrides can also handle exceptions within the same organization.

This keeps the credential-selection logic entirely inside `.gitconfig`, without storing PATs directly or requiring external scripts.
