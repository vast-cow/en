---
title: "Getting ytmusicapi Headers with Headless Chrome"
description: "This script collects the request information needed to use ytmusicapi with YouTube Music. It connects..."
pubDatetime: 2026-05-01T13:50:32.748Z
---

This script collects the request information needed to use `ytmusicapi` with YouTube Music. It connects to an already running Chrome or Edge browser, opens YouTube Music, watches the network traffic, and saves the relevant request headers to a JSON file.

## Purpose

`ytmusicapi` may require header and authentication-related information from an active YouTube Music session.

This script helps collect that information by using a real browser session. It listens for requests sent to the YouTube Music internal API and captures the headers from the matching request.

The target request is:

```text
https://music.youtube.com/youtubei/v1/browse
```

When the script finds a matching request, it saves the details to:

```text
matched_request_headers.json
```

## What the Script Does

The script performs the following steps:

1. Connects to an existing Chrome or Edge browser through the Chrome DevTools Protocol.
2. Selects an existing browser tab.
3. Adjusts the User-Agent so it looks like a normal Chrome browser.
4. Opens the YouTube Music uploads library page.
5. Monitors network requests from the browser.
6. Captures only requests that match the YouTube Music `browse` API endpoint.
7. Saves the request URL, method, resource type, and headers to a JSON file.
8. Returns the tab to the new tab page after collection.

## Preparation

Before running the script, start Chrome or Edge with remote debugging enabled.

For example:

```bash
chrome --remote-debugging-port=9222
```

Depending on your operating system, you may need to use the full path to the Chrome or Edge executable.

You also need Playwright installed in your Python environment:

```bash
pip install playwright
playwright install
```

## How to Use It

First, open Chrome or Edge with the remote debugging port enabled.

Next, sign in to YouTube Music in that browser. Since the script connects to the existing browser session, it can use your logged-in YouTube Music state.

Then run the script:

```bash
python script.py
```

The script will connect to the browser and navigate to:

```text
https://music.youtube.com/library/uploads
```

While the page loads, it monitors network requests. If it finds a request matching the target API endpoint, it prints the request details in the console and saves them to:

```text
matched_request_headers.json
```

## Output

The output JSON file contains information like this:

```json
{
  "effective_user_agent": "...",
  "matched_count": 1,
  "matched_requests": [
    {
      "url": "...",
      "url_without_query": "https://music.youtube.com/youtubei/v1/browse",
      "method": "POST",
      "resource_type": "xhr",
      "headers": {
        "...": "..."
      }
    }
  ]
}
```

The most important part is the `headers` object inside `matched_requests`. This contains the request headers that can be used when preparing `ytmusicapi` authentication or configuration data.

## Why the User-Agent Is Adjusted

When Chrome runs in headless mode, its User-Agent may include the string `HeadlessChrome`.

The script replaces `HeadlessChrome` with `Chrome` so the browser identifies itself more like a regular Chrome browser.

This helps keep the request environment closer to a normal browser session.

## How Matching Works

The script does not save every network request.

Instead, it removes query parameters from each request URL and checks whether the remaining URL exactly matches:

```text
https://music.youtube.com/youtubei/v1/browse
```

This keeps the output focused and avoids saving unrelated requests such as images, scripts, stylesheets, and other API calls.

## Notes

The script connects to an existing browser, so it does not close the browser when it finishes.

After collecting the request data, it tries to return the selected tab to:

```text
chrome://new-tab-page
```

If no matching request is found, `matched_count` will be `0`. In that case, check that you are signed in to YouTube Music and that the uploads library page loaded correctly.

## Summary

This script is a small helper tool for collecting YouTube Music request headers for use with `ytmusicapi`.

It uses Playwright to connect to an existing Chrome or Edge session, opens YouTube Music, watches for the relevant internal API request, and saves the matched request headers to `matched_request_headers.json`. Because it works with a real logged-in browser session, the captured headers reflect the actual YouTube Music environment used by the browser.

```python
import json
import urllib.request
from urllib.parse import urlparse
from playwright.sync_api import sync_playwright, Request, Page


# ===== Configuration =====

CDP_ENDPOINT = "http://127.0.0.1:9222"  # Existing browser CDP port
URL_A = "https://music.youtube.com/library/uploads"

# Condition B:
# Match requests whose URL, excluding query parameters, is exactly:
# https://music.youtube.com/youtubei/v1/browse
URL_B_BASE = "https://music.youtube.com/youtubei/v1/browse"

OUTPUT_JSON = "matched_request_headers.json"


def assert_cdp_available(endpoint: str) -> None:
    version_url = endpoint.rstrip("/") + "/json/version"

    try:
        with urllib.request.urlopen(version_url, timeout=3) as res:
            if res.status != 200:
                raise RuntimeError(f"CDP endpoint returned HTTP {res.status}")
    except Exception as e:
        raise RuntimeError(
            f"CDP endpoint is not available: {version_url}\n"
            f"Please start Chrome/Edge with --remote-debugging-port=9222.\n"
            f"Original error: {e}"
        ) from e


def get_url_without_query(url: str) -> str:
    """
    Return the URL without query parameters or fragments.

    Example:
      https://music.youtube.com/youtubei/v1/browse?key=abc
      -> https://music.youtube.com/youtubei/v1/browse
    """
    parsed = urlparse(url)
    return f"{parsed.scheme}://{parsed.netloc}{parsed.path}"


def matches_condition_b(url: str) -> bool:
    """
    Condition B:
    Match only when the URL without query parameters is exactly URL_B_BASE.
    """
    return get_url_without_query(url) == URL_B_BASE


def pick_existing_page(context) -> Page:
    """
    Pick one existing tab.

    Prefer an chrome://new-tab-page tab if available.
    Otherwise, use the first existing tab.
    """
    pages = context.pages

    if not pages:
        raise RuntimeError(
            "No existing tab was found. Please open at least one tab in the CDP-connected browser."
        )

    for page in pages:
        if page.url == "chrome://new-tab-page":
            return page

    return pages[0]


def get_default_user_agent(page: Page) -> str:
    """
    Read navigator.userAgent from the current page environment.

    Using the actual User-Agent from the CDP-connected browser avoids mismatch
    between the real Chrome version and the User-Agent string.
    """
    user_agent = page.evaluate("navigator.userAgent")

    if not isinstance(user_agent, str) or not user_agent.strip():
        raise RuntimeError("Failed to read navigator.userAgent")

    return user_agent


def normalize_user_agent(user_agent: str) -> str:
    """
    Replace HeadlessChrome with Chrome.

    If the User-Agent does not contain HeadlessChrome, return it unchanged.
    """
    return user_agent.replace("HeadlessChrome", "Chrome")


def get_default_platform(page: Page) -> str:
    """
    Read navigator.platform from the current page environment.

    Return an empty string if it cannot be read.
    """
    try:
        platform = page.evaluate("navigator.platform")
    except Exception:
        return ""

    if not isinstance(platform, str):
        return ""

    return platform


def spoof_user_agent(context, page: Page) -> str:
    """
    Override the User-Agent for an existing CDP-connected browser page.

    For an existing browser connected through CDP, use CDP's
    Network.setUserAgentOverride instead of new_context(user_agent=...).

    Steps:
      1. Read the current navigator.userAgent.
      2. Replace HeadlessChrome with Chrome.
      3. Apply the value using Network.setUserAgentOverride.

    Returns:
      The actual User-Agent value that was applied.
    """
    default_user_agent = get_default_user_agent(page)
    override_user_agent = normalize_user_agent(default_user_agent)
    platform = get_default_platform(page)

    print(f"Default User-Agent : {default_user_agent}")
    print(f"Override User-Agent: {override_user_agent}")
    print(f"Platform           : {platform}")

    cdp_session = context.new_cdp_session(page)

    cdp_session.send("Network.enable")

    params = {
        "userAgent": override_user_agent,
    }

    if platform:
        params["platform"] = platform

    cdp_session.send("Network.setUserAgentOverride", params)

    return override_user_agent


def main() -> None:
    matched_requests = []

    assert_cdp_available(CDP_ENDPOINT)

    with sync_playwright() as p:
        browser = p.chromium.connect_over_cdp(CDP_ENDPOINT)

        if not browser.contexts:
            raise RuntimeError("No existing browser context was found.")

        context = browser.contexts[0]
        page = pick_existing_page(context)

        # Apply User-Agent spoofing
        effective_user_agent = spoof_user_agent(context, page)

        def handle_request(request: Request) -> None:
            url = request.url

            if not matches_condition_b(url):
                return

            try:
                headers = request.all_headers()

                record = {
                    "url": url,
                    "url_without_query": get_url_without_query(url),
                    "method": request.method,
                    "resource_type": request.resource_type,
                    "headers": headers,
                }

                matched_requests.append(record)

                print("=== MATCHED REQUEST ===")
                print(json.dumps(record, ensure_ascii=False, indent=2))

            except Exception as e:
                print(f"[WARN] Failed to read headers for {url}: {e}")

        context.on("request", handle_request)

        try:
            # Navigate the existing tab to URL A
            page.goto(URL_A, wait_until="domcontentloaded")

            try:
                page.wait_for_load_state("networkidle", timeout=15_000)
            except Exception:
                pass

            output = {
                "effective_user_agent": effective_user_agent,
                "matched_count": len(matched_requests),
                "matched_requests": matched_requests,
            }

            with open(OUTPUT_JSON, "w", encoding="utf-8") as f:
                json.dump(output, f, ensure_ascii=False, indent=2)

            print(f"Saved: {OUTPUT_JSON}")
            print(f"Matched count: {len(matched_requests)}")

        finally:
            # After collection, return the existing tab to the new tab page
            try:
                page.goto("chrome://new-tab-page", wait_until="domcontentloaded")
            except Exception as e:
                print(f"[WARN] Failed to navigate tab to chrome://new-tab-page: {e}")

            # Do not close the existing browser
            # browser.close()


if __name__ == "__main__":
    main()
```
