---
title: "Git Alias to Set Commit User Information Using Only Your GitHub Username"
description: "This script retrieves user information from the GitHub REST API and defines a Git alias in the .gitconfig file to set a no-reply email address for the current repository."
pubDatetime: 2026-10-07T12:53:00+09:00
---

When working with multiple GitHub accounts, setting the author information for commits in each repository can be a bit tedious.

```bash
git config --local user.name "your-username"
git config --local user.email "12345678+your-username@users.noreply.github.com"
```

In particular, looking up the GitHub numeric user ID each time is cumbersome. So, I created a Git alias that **sets `user.name` and `user.email` in the current repository just by passing the GitHub username.**

Because it is defined directly in `~/.gitconfig`, there is no need to prepare a separate script file.

## What Does This Alias Do?

For example, run the following in a Git repository:

```bash
git config-gh octocat
```

The alias retrieves the user's `id`, `login`, and `name` from the GitHub REST API and sets them in the following format:

```text
user.name  = The Octocat
user.email = 583231+octocat@users.noreply.github.com
```

For `user.name`, the display name from the GitHub profile (`name`) is used; if not set, the username (`login`) is used.

Also, because `git config --local` is used, the **changes to the user information for commits are only applied to the current repository**. The global `user.name` and `user.email` are not changed.

## Requirements

- Git
- `curl`
- `jq`
- POSIX-compatible shell (`sh`)

Because the Git alias uses the shell execution function, you do not need to explicitly invoke Bash.

Also, there is no need to install `gh` (GitHub CLI) or switch login accounts with `gh auth switch`.

## Configuring the Git Alias

Add the following to the `[alias]` section of `~/.gitconfig`:

```gitconfig
[alias]
    config-gh = "!f() { \
        if [ \"$#\" -ne 1 ] || [ -z \"$1\" ]; then \
            printf 'Usage: git config-gh <github-username>\\n' >&2; return 1; \
        fi; \
        username=$1; \
        case \"$username\" in \
            -*|*-|*[!A-Za-z0-9-]*|*--*) \
                printf 'Invalid GitHub username: %s\\n' \"$username\" >&2; return 1 ;; \
        esac; \
        if [ \"${#username}\" -gt 39 ]; then \
            printf 'Invalid GitHub username: %s\\n' \"$username\" >&2; return 1; \
        fi; \
        for cmd in curl jq git; do \
            if ! command -v \"$cmd\" >/dev/null 2>&1; then \
                printf 'Required command not found: %s\\n' \"$cmd\" >&2; return 1; \
            fi; \
        done; \
        if ! git rev-parse --is-inside-work-tree >/dev/null 2>&1; then \
            printf 'Run this command inside a Git working tree.\\n' >&2; return 1; \
        fi; \
        if ! response=$(curl --fail --silent --show-error --location \
            --connect-timeout 5 --max-time 15 \
            \"https://api.github.com/users/${username}\"); then \
            printf 'Failed to fetch GitHub user: %s\\n' \"$username\" >&2; return 1; \
        fi; \
        if ! user_info=$(printf '%s' \"$response\" | jq -er ' \
            select(.id | type == \"number\") \
            | select(.login | type == \"string\" and length > 0) \
            | [.id, .login, (.name // .login)] \
            | @tsv \
        '); then \
            printf 'Unexpected GitHub API response for: %s\\n' \"$username\" >&2; return 1; \
        fi; \
        tab=$(printf '\\t'); \
        id=${user_info%%\"$tab\"*}; \
        user_info=${user_info#*\"$tab\"}; \
        login=${user_info%%\"$tab\"*}; \
        name=${user_info#*\"$tab\"}; \
        email=\"${id}+${login}@users.noreply.github.com\"; \
        git config --local user.name \"$name\" || return 1; \
        git config --local user.email \"$email\" || return 1; \
        printf 'user.name  = %s\\n' \"$(git config --local user.name)\"; \
        printf 'user.email = %s\\n' \"$(git config --local user.email)\"; \
    }; f"
```

The `!` at the beginning of the alias specifies that it should be executed as a shell command rather than as a Git subcommand.

```sh
!f() { ...; }; f
```

In this format, the shell function `f` is defined and executed immediately.

Additional arguments passed to the Git alias are also passed to the shell function, so you can get the GitHub username with `$1`.

The processing is written in POSIX `sh`-compatible syntax, so `bash -c` is not required.

Note that double quotes and backslashes must be escaped in `.gitconfig`, so it is different from a normal shell script and uses notations like `\"` and `\\n`.

## How to Use It

Go to the Git repository you want to configure, and run it by specifying the GitHub username.

```bash
cd /path/to/repository
git config-gh octocat
```

To check the configured information, use the following command:

```bash
git config --local --get user.name
git config --local --get user.email
```

You can check if the alias is registered with the following command:

```bash
git config --global --get alias.config-gh
```

Because it is defined directly in `~/.gitconfig`, there is no need to create a script file, grant execution permissions, or set up the `PATH`.

## How It Works

### 1. Execute the Shell Function from the Git Alias

Git has a mechanism where you can execute an arbitrary shell command by starting the alias definition with `!`.

For example, you can define an alias like this:

```gitconfig
[alias]
    hello = "!f() { echo \"Hello, $1\"; }; f"
```

When executed, the argument is passed to the shell function.

```bash
git hello octocat
# Hello, octocat
```

The `config-gh` uses the same mechanism.

### 2. Validate the Arguments and Execution Environment

First, it checks that only one argument is specified.

Then, using `case` and a length check, it excludes invalid strings as GitHub usernames.

Usernames must consist of alphanumeric characters and hyphens, and hyphens cannot be used at the beginning and end. Consecutive hyphens and strings longer than 39 characters are also rejected.

Finally, it checks that `curl`, `jq`, and `git` are executable and that the current directory is within a Git working tree.

### 3. Obtain User Information from the GitHub REST API

GitHub has an API that allows you to retrieve public profiles from a username.

```bash
curl -fsSL https://api.github.com/users/octocat \
  | jq '{id, login, name}'
```

`id` is the numeric user ID within GitHub, `login` is the username, and `name` is the profile display name.

The alias uses `jq` to extract the necessary information.

```jq
select(.id | type == "number")
| select(.login | type == "string" and length > 0)
| [.id, .login, (.name // .login)]
| @tsv
```

If `name` is `null`, `login` is used.

The extracted information is output with tab delimiters, and the shell parameter expansion is used to split it into `id`, `login`, and `name`.

### 4. Assemble GitHub's noreply Email Address

From the obtained information, the email address in the following format is assembled:

```text
<ID>+<USERNAME>@users.noreply.github.com
```

This is one of the private email address formats provided by GitHub.

The actual noreply address that can be used may vary depending on the timing of the account creation and email privacy settings. Check in GitHub's "Settings → Emails."

### 5. Modify the Local Settings of the Repository

The profile display name and the generated email address are used to modify the settings of the current repository.

```sh
git config --local user.name "$name"
git config --local user.email "$email"
```

`--local` is an option to modify the settings of the current repository.

Although the alias definition itself is in `~/.gitconfig`, the changes to `user.name` and `user.email` are applied to the local settings of the current repository.

It does not affect other repositories or the global commit user information.

## Notes

### The GitHub Authentication Account Does Not Change

This alias only changes the author name and email address that are recorded in the Git commit.

The SSH key, authentication token, or GitHub CLI login state used when running `git push` is not changed.

Also, the author information of previously created commits is not rewritten.

### Check the noreply Email Address

Even if you assemble the GitHub noreply email address, it does not necessarily mean that it is valid for that account.

Depending on the timing of account creation and email privacy settings, a different format of noreply address may be used.

To correctly associate commits with the GitHub account, use the email address that is actually associated with the account.

### GitHub API Rate Limiting

Because you call the GitHub REST API without authentication, there is a rate limit.

It is not suitable for frequent or massive execution. Also, it will exit with an error if there is a network failure or if the username does not exist.

### `user.name` Uses the Profile Display Name

`user.name` uses the GitHub `name`, and if not set, falls back to `login`.

If you always want to set the GitHub username, change the following part of `jq`:

```jq
[.id, .login, (.name // .login)]
```

Change to:

```jq
[.id, .login, .login]
```

This will always set the GitHub `login` to `user.name`.

## Summary

If you know the GitHub username, you can use the REST API to retrieve the numeric user ID and profile information and set the commit user information for the current repository.

Because it can be defined directly in `~/.gitconfig` as a Git alias, there is no need to manage external scripts or set execution permissions.

Also, it is an advantage that you do not need to switch authentication accounts with `gh auth switch` and can maintain the usual authentication state.

It can simplify the configuration process for each repository in development environments that use multiple GitHub accounts.

References:

- [Git — git-config Documentation](https://git-scm.com/docs/git-config)
- [GitHub REST API — Get a user](https://docs.github.com/en/rest/users/users#get-a-user)
- [GitHub — Setting your commit email address](https://docs.github.com/en/account-and-profile/how-tos/email-preferences/setting-your-commit-email-address)
- [GitHub — Email addresses reference](https://docs.github.com/en/account-and-profile/reference/email-addresses-reference)
