---
title: `git-config-gh` Function for Managing Multiple GitHub Accounts
description: "This Bash function leverages `gh` (GitHub CLI) to switch between GitHub accounts and retrieve user information from the GitHub API. It then automatically configures the current repository with a \"noreply\" email address."
pubDatetime: 2026-10-07T12:53:00+09:00
---

If you use multiple GitHub accounts (e.g., for work and personal use) on a single computer, you may find yourself needing to switch between `git config user.name` and `git config user.email` for each repository.

Especially if you're using GitHub's email privacy settings, you can set a "noreply" address for your commits, such as:

```text
12345678+username@users.noreply.github.com
```

Manually setting this every time can be tedious.

Therefore, I created a Bash function called `git-config-gh` that uses the GitHub CLI (`gh`) to temporarily switch to the desired account, retrieves user information from the GitHub API, and configures the current Git repository with the "noreply" email address using the `--local` setting.

## Final Form

```bash
git-config-gh() (
  local username="$1"
  local original
  local login
  local id

  if [[ -z "$username" ]]; then
    echo "usage: git-config-gh <github-username>" >&2
    return 1
  fi

  original="$(gh api user --jq '.login')" || return 1

  # Restore the original account regardless of success or failure
  trap 'gh auth switch --hostname github.com --user "$original" >/dev/null 2>&1 || true' EXIT

  gh auth switch --hostname github.com --user "$username" || return 1

  read -r id login < <(
    gh api user --jq '"\(.id) \(.login)"'
  ) || return 1

  git config --local user.name "$login"
  git config --local user.email "${id}+${login}@users.noreply.github.com"

  echo "user.name  = $(git config --local user.name)"
  echo "user.email = $(git config --local user.email)"
)
```

For example, to set the GitHub account "alice" for the current repository, run:

```bash
git-config-gh alice
```

After execution, the following settings will be added to the `.git/config` file of the current repository:

```ini
[user]
    name = alice
    email = 12345678+alice@users.noreply.github.com
```

## Prerequisites

This function utilizes the GitHub CLI, `gh`.

Also, the GitHub account you want to use must already be logged in to `gh`.

You can check the login status with the following command:

```bash
gh auth status
```

If you have multiple accounts registered, you can switch between active accounts using `gh auth switch`.

```bash
gh auth switch --user alice
```

This function automates this switching process internally.

## Processing Flow

When you run `git-config-gh alice`, the following process generally occurs:

```text
Get the current gh account
        ↓
Switch to alice with gh auth switch
        ↓
Retrieve ID and login from the GitHub API
        ↓
Set git config --local user.name
        ↓
Set git config --local user.email
        ↓
Return to the original gh account
```

The key is that only the Git configuration is changed, and the active account on the `gh` side is ultimately returned to its original state.

## Saving the Current GitHub Account

First, the currently active account in `gh` is retrieved.

```bash
original="$(gh api user --jq '.login')" || return 1
```

`gh api user` retrieves information about the GitHub user currently used for authentication.

By specifying `--jq '.login'`, only the GitHub username is extracted, not the entire response.

For example, if the current account is `bob`,

```text
bob
```

is saved in `original`.

## Switching to the Target Account

Next, the GitHub account specified as an argument is switched to.

```bash
gh auth switch --hostname github.com --user "$username" || return 1
```

For example, if you run

```bash
git-config-gh alice
```

the following operation is essentially performed:

```bash
gh auth switch --hostname github.com --user alice
```

Because the account is switched here, the subsequent `gh api user` will return information for `alice`.

## Obtaining the GitHub User ID and Login

The GitHub "noreply" email address uses not only the username but also a numerical user ID.

Therefore, both are retrieved from the API.

```bash
read -r id login < <(
  gh api user --jq '"\(.id) \(.login)"'
)
```

For example, if the information on the API is:

```json
{
  "login": "alice",
  "id": 12345678
}
```

the following values are set:

```text
id=12345678
login=alice
```

These two are then used to construct the email address:

```text
12345678+alice@users.noreply.github.com
```

## Setting `git config --local`

The retrieved information is then configured for the current repository.

```bash
git config --local user.name "$login"
git config --local user.email "${id}+${login}@users.noreply.github.com"
```

The `--local` option is explicitly specified here.

Therefore, only the `.git/config` file of the current Git repository is modified.

The global settings:

```bash
git config --global user.name
git config --global user.email
```

are not affected.

When using multiple GitHub accounts, this repository-specific configuration is less prone to errors.

## Restoring the Original Account on Exit

The `trap` command is crucial in this function.

```bash
trap 'gh auth switch --hostname github.com --user "$original" >/dev/null 2>&1 || true' EXIT
```

Regardless of how the function terminates, it restores the original GitHub account on `EXIT`.

This restore process runs not only on normal termination but also if the function encounters an error and `return 1` is called.

For example:

```text
bob is active
↓
git-config-gh alice
↓
Switch to alice
↓
Git configuration
↓
Return to bob
```

is the resulting behavior.

## Why the Function is Enclosed in `()`

Normal Bash functions are often written as:

```bash
foo() {
  ...
}
```

using `{}`. However, in this case, the function is written as:

```bash
git-config-gh() (
  ...
)
```

The code within `()` is executed in a subshell.

This allows you to isolate the `trap` and local execution environment within the function from the calling shell.

This is a convenient way to write functions that use `trap ... EXIT`.

## Registering in `.bashrc` or `.zshrc`

If you use this function regularly, add it to your shell configuration file.

For Bash, add it to:

```bash
~/.bashrc
```

If you use Zsh on macOS, add it to:

```bash
~/.zshrc
```

After reflecting the changes, you can run:

```bash
git-config-gh alice
```

## Verifying the Configuration

The function displays the set values at the end.

```bash
echo "user.name  = $(git config --local user.name)"
echo "user.email = $(git config --local user.email)"
```

For example:

```text
user.name  = alice
user.email = 12345678+alice@users.noreply.github.com
```

is displayed.

You can also verify manually using the following commands:

```bash
git config --local user.name
git config --local user.email
```

or

```bash
git config --local --list
```

## Summary

With `git-config-gh`, you can switch the commit user for the current repository with a single command:

```bash
git-config-gh alice
```

when using multiple GitHub accounts.

The key points of this function are:

- Temporarily switches to the target GitHub account using `gh auth switch`.
- Retrieves the user ID and login from the GitHub API.
- Automatically generates `ID+login@users.noreply.github.com`.
- Only modifies the `git config --local` settings.
- Returns to the original `gh` account upon function termination.
- Does not require explicitly handling `GH_TOKEN`.

If you use separate GitHub accounts for work and personal use, this is a simple yet useful helper function.