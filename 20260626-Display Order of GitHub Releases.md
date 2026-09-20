---
title: "Display Order of GitHub Releases"
description: "Examine how latest-release selection, semantic versions, and commit dates affect GitHub releases, and sort API results explicitly for predictable ordering."
pubDatetime: 2026-06-26T12:22:25.815Z
---

## Conclusion

The display order of GitHub Releases is **not based on the release title**, **not based on the tag name in simple lexical order**, and **not simply sorted by publication date in descending order**.

If you want more predictable ordering for GitHub Releases, it is recommended to use **SemVer-formatted tag names**.

GitHub determines release ordering using a combination of the following criteria:

| Aspect                 | What is Used                                                                                                                      |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| "Latest" determination | An explicitly selected `Set as latest release` / `make_latest`, or automatic SemVer-based detection                               |
| `/releases/latest`     | The release considered "latest," excluding drafts and prereleases                                                                 |
| Releases page display  | Influenced by tag/commit dates and SemVer rather than publication date. The exact sorting algorithm is not officially documented. |

In the GitHub UI, you can choose **Set as latest release** when creating a release. If you do not, the `latest` label is automatically assigned based on Semantic Versioning, according to the official documentation. ([GitHub Docs][1])

## Why Doesn't It Appear to Be Sorted by Date?

According to the GitHub REST API documentation for `Get the latest release`, the latest release is determined using `created_at`. However, `created_at` does **not** represent when the release was drafted or published—it represents the **date of the commit associated with the release**. ([GitHub Docs][2])

For example:

```text
Release A published on 2026-06-26
  -> The tag points to an old commit from 2025

Release B published on 2026-06-20
  -> The tag points to a newer commit from 2026
```

In this case, although Release A has the newer publication date, GitHub may still treat Release B as the newer release.

## Is It Sorted by Tag Name?

**No, not in simple lexical order.**

However, if the tag name can be interpreted as a Semantic Version, it does influence GitHub's ordering.

The official GitHub Changelog explains that, before explicit "latest" selection was introduced, the latest release was determined by the most recent release date, with Semantic Versioning used as a tie-breaker when releases shared the same date. GitHub now allows maintainers to explicitly designate the latest release. ([The GitHub Blog][3])

A GitHub Community discussion also explains that releases created on the same day are expected to be ordered by SemVer, and that GitHub primarily relies on **tag/commit dates** rather than the release object's publication timestamp. ([GitHub][4])

As a result, tag names like the following can produce unexpected ordering:

```text
release/69.3
build-20240626
2024-06-26-001
v1.0
v1.0.0-beta.10
v1.0.0-beta.9
```

If you want GitHub to recognize version ordering correctly, use standard SemVer tags such as:

```text
v1.2.3
v1.2.4
v1.3.0
v2.0.0
```

## Is It Sorted by Title?

**No.**

The release title (`name`) is purely a display field. In the GitHub API, `tag_name` and `name` are separate fields. For ordering and latest-release determination, the relevant fields are the tag, SemVer, `created_at`, `published_at`, and `make_latest`—not the release title. ([GitHub Docs][5])

## Notes When Using the API

The `GET /repos/{owner}/{repo}/releases` endpoint (List releases) provides pagination parameters such as `per_page` and `page`, but it does **not** support explicit sorting parameters such as `sort` or `direction`. ([GitHub Docs][5])

GitHub Community discussions also note that the API does not guarantee a particular ordering, so applications should not depend on the response order. ([GitHub][6])

Therefore, if your application requires deterministic ordering, you should sort the releases yourself.

```bash
gh release list \
  --repo OWNER/REPO \
  --limit 100 \
  --json tagName,name,publishedAt,createdAt,isLatest,isPrerelease,isDraft \
  --jq '.[] | [.tagName, .name, .publishedAt, .createdAt, .isLatest, .isPrerelease, .isDraft] | @tsv'
```

`gh release list` can output fields such as `createdAt`, `publishedAt`, `tagName`, `name`, `isLatest`, `isPrerelease`, and `isDraft` as JSON. ([GitHub CLI][7])

## Practical Recommendations

If you want your releases to appear in a predictable order, the following practices are recommended:

1. **Use SemVer-formatted tags**

   For example:

   ```text
   v1.2.3
   ```

2. **Avoid creating releases for old commits after newer releases**

   GitHub's ordering can be influenced by the tag's commit date rather than the publication date.

3. **Explicitly mark the release as the latest when appropriate**

   In the UI, select **Set as latest release**. Via the API, use `make_latest: true`. The `make_latest` field accepts `true`, `false`, or `legacy`, and draft or prerelease releases cannot be marked as the latest. ([GitHub Docs][2])

4. **Do not rely on GitHub's response order**

   Sort releases yourself according to your application's requirements—for example, by `publishedAt`, `createdAt`, or Semantic Version.

In summary, the order of GitHub Releases is determined **neither by the release title, nor by the tag name in simple lexical order, nor by publication date alone**. Instead, it is influenced by GitHub's internal handling of the latest release, Semantic Versioning, and tag/commit dates.

[1]: https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository "Managing releases in a repository - GitHub Docs"
[2]: https://docs.github.com/rest/releases/releases "REST API endpoints for releases - GitHub Docs"
[3]: https://github.blog/changelog/2022-10-20-explicitly-set-the-latest-release/ "Explicitly Set the Latest Release - GitHub Changelog"
[4]: https://github.com/orgs/community/discussions/8226 "Releases out of order · community · Discussion #8226 · GitHub"
[5]: https://docs.github.com/ja/rest/releases/releases "REST API endpoints for releases - GitHub Docs"
[6]: https://github.com/orgs/community/discussions/21901 "Releases API - can't figure out order · community · Discussion #21901 · GitHub"
[7]: https://cli.github.com/manual/gh_release_list?utm_source=chatgpt.com "gh release list"
