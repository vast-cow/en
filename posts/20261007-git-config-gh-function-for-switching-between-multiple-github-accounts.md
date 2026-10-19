---
title: "Shell script to set Git commit user information using only the GitHub username"
description: "This Bash retrieve user information from the GitHub API. It then automatically configures the current repository with a \"noreply\" email address."
pubDatetime: 2026-10-07T12:53:00+09:00
---

If you use multiple GitHub accounts, setting the author information for commits for each repository can be a bit cumbersome.

```bash
git config --local user.name "your-username"
git config --local user.email "12345678+your-username@users.noreply.github.com"
```

In particular, it is time-consuming to look up the GitHub numerical user ID each time. Therefore, I created a script that **sets `user.name` and `user.email` in the current repository simply by passing the GitHub username.**

## What does this script do?

For example, run the following command inside a Git repository:

```bash
./git-config-gh.sh octocat
```

The script retrieves the user's `id` and `login` from the GitHub REST API and sets them in the following format:

```text
user.name  = octocat
user.email = <numerical ID>+octocat@users.noreply.github.com
```

Because it uses `git config --local`, it applies **only to the current repository** and does not change the global settings.

## Requirements

- Bash
- `curl`
- `jq`
- Git

It is not necessary to install `gh` (GitHub CLI) or switch login accounts using `gh auth switch`.

## Script

Save the following as `git-config-gh.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
  printf 'Usage: %s <github-username>\n' "${0##*/}" >&2
}

if [[ $# -ne 1 || -z "$1" ]]; then
  usage
  exit 1
fi

username="$1"
if [[ ! "$username" =~ ^[A-Za-z0-9]([A-Za-z0-9-]{0,37}[A-Za-z0-9])?$ ]]; then
  printf 'Invalid GitHub username: %s\n' "$username" >&2
  exit 1
fi

for command in curl jq git; do
  if ! command -v "$command" >/dev/null 2>&1; then
    printf 'Required command not found: %s\n' "$command" >&2
    exit 1
  fi
done

if ! git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  printf 'Run this command inside a Git working tree.\n' >&2
  exit 1
fi

if ! response="$(curl --fail --silent --show-error --location \
  --connect-timeout 5 --max-time 15 \
  "https://api.github.com/users/${username}")"; then
  printf 'Failed to fetch GitHub user: %s\n' "$username" >&2
  exit 1
fi

if ! user_info="$(jq -er '
  select(.id | type == "number")
  | select(.login | type == "string" and length > 0)
  | [.id, .login, (.name // .login)]
  | @tsv
' <<< "$response")"; then
  printf 'Unexpected GitHub API response for: %s\n' "$username" >&2
  exit 1
fi

IFS=$'\t' read -r id login name <<< "$user_info"
email="${id}+${login}@users.noreply.github.com"

git config --local user.name "$name"
git config --local user.email "$email"

printf 'user.name  = %s\n' "$(git config --local user.name)"
printf 'user.email = %s\n' "$(git config --local user.email)"
```

## Usage

Give execute permission:

```bash
chmod +x git-config-gh.sh
```

Run inside the repository you want to configure (if you saved the script in a different location, specify that path):

```bash
cd /path/to/repository
/path/to/git-config-gh.sh octocat
```

To confirm the configuration, use the following command:

```bash
git config --local --get user.name
git config --local --get user.email
```

If you use it frequently, it is convenient to put the script in a location with PATH, such as `~/bin/git-config-gh`.

## How it works

### 1. Get user information from the GitHub REST API

GitHub has an API that allows you to retrieve a public profile from a username.

```bash
curl -fsSL https://api.github.com/users/octocat | jq '{id, login}'
```

`id` is the numerical user ID internal to GitHub, and `login` is the username. The script extracts these two from the JSON.

### 2. Assemble the GitHub noreply email address

From the acquired information, assemble the email address in the following format:

```text
<ID>+<USERNAME>@users.noreply.github.com
```

This is one of the formats that GitHub uses for private email addresses. Verify that this matches the actual noreply address associated with your GitHub account in the email settings.

### 3. Rewrite the local settings of the repository

```bash
git config --local user.name "$login"
git config --local user.email "$email"
```

`--local` changes the settings for the current repository. It does not affect other repositories or `~/.gitconfig`.

## Notes

**The authenticated account for GitHub will not be switched.** This script only changes the author name and email address recorded in Git commits. The SSH key, authentication token, or GitHub CLI login state used for `git push` will not be changed.

Also, **even if you assemble the GitHub noreply email address, it is not guaranteed to be valid for that account.** Please confirm the actual address in "Settings → Emails" on GitHub. The association of commits to GitHub and email address privacy settings depend on the account settings.

Because the GitHub API is called without authentication, there is a rate limit. It is not suitable for frequent, large-scale execution. In addition, it will exit with an error if there is a network failure or the specified user does not exist.

Note that `user.name` is set to the GitHub `login`. It does not set the display name (`name`) in the profile.

## Summary

If you know the GitHub username, you can use the REST API to retrieve the numerical ID and set the commit user information for the current repository.

The advantage is that you do not need to switch accounts with `gh auth switch` and you can maintain your usual authentication status. It simplifies the task of setting up repositories on a per-repository basis in development environments that handle multiple accounts.

References: [GitHub REST API — Get a user](https://docs.github.com/en/rest/users/users#get-a-user) / [GitHub — Setting your commit email address](https://docs.github.com/en/account-and-profile/how-tos/email-preferences/setting-your-commit-email-address)
