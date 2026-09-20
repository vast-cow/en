---
title: "Extracting a Shared Box URL from the Login Page"
description: "Recover a Box destination from the login page's redirect parameter and copy the reconstructed sharing URL to the clipboard with a browser bookmarklet."
pubDatetime: 2026-01-19T10:23:36.831Z
---

When you open a Box file link on Windows, your browser may redirect you to the Box login page instead of taking you directly to the shared content. In many cases, the login page URL includes a query parameter that contains the original destination. If you need to quickly recover the shareable link (for example, to send it to someone or open it in another environment), it is useful to extract that destination URL from the login page.

The following JavaScript snippet is designed for that purpose. It reads the current login page URL, retrieves the `redirect_url` parameter from the query string, reconstructs the full destination URL, and copies it to your clipboard automatically.

```javascript
(function () {
  const hostOrig = location.host;

  // Skip if host does not end with box.com
  if (!hostOrig.endsWith('box.com')) return;

  const usp = new URLSearchParams(location.search);
  const path = usp.get('redirect_url');

  // Skip if redirect_url is missing or empty
  if (!path) return;

  const host = hostOrig
    .split('.')
    .filter((part) => part !== 'account')
    .join('.');

  // If redirect_url is an absolute URL, use it as-is; otherwise build https://{host}{path}
  const url = 'https://' + host + path;

  const tempTextarea = document.createElement('textarea');
  tempTextarea.textContent = url;
  document.body.appendChild(tempTextarea);
  tempTextarea.select();
  document.execCommand('copy');
  document.body.removeChild(tempTextarea);
})();
```

```javascript
javascript:!function(){const hostOrig=location.host;if(!hostOrig.endsWith("box.com"))return;const path=new URLSearchParams(location.search).get("redirect_url");if(!path)return;const url="https://"+hostOrig.split(".").filter(part=>"account"!==part).join(".")+path;const tempTextarea=document.createElement("textarea");tempTextarea.textContent=url;document.body.appendChild(tempTextarea);tempTextarea.select();document.execCommand("copy");document.body.removeChild(tempTextarea)}();
```

Here is what the script does, step by step:

* **Parses the query string** using `URLSearchParams` to access parameters on the current page.
* **Extracts `redirect_url`** to obtain the path (or full redirect target) that Box stored when it sent you to the login flow.
* **Rebuilds the host name** by removing the `account` subdomain (common on Box login pages) so the destination points to the main Box domain.
* **Constructs the final URL** by combining `https://`, the adjusted host, and the redirect path.
* **Copies the result to the clipboard** by temporarily creating a `<textarea>`, selecting its contents, and executing the copy command.
* **Cleans up** by removing the temporary element from the page.

In practice, this provides a quick way to recover a share-ready URL directly from the Box login screen, without manually decoding or editing the address bar.
