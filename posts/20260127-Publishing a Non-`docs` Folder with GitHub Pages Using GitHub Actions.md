---
title: "Publishing a Non-`docs` Folder with GitHub Pages Using GitHub Actions"
description: "Deploy an arbitrary static-site directory through GitHub Actions by configuring Pages permissions, path-based triggers, artifact upload, and the deployment environment."
pubDatetime: 2026-01-27T12:38:02.646Z
---

GitHub Pages is often set up to publish from common locations like `/docs` or a dedicated branch. However, if you want to publish a different folder—such as `frontend/`—a reliable approach is to use **GitHub Actions** for deployment. This avoids mismatches that can happen when the repository is still configured for “Deploy from a branch.”

## Configure GitHub Pages to Use GitHub Actions

Start by updating your repository’s Pages settings:

1. Open your repository and go to **Settings** → **Pages**.
2. Find the **Build and deployment** section.
3. Set **Source** to **GitHub Actions**.

If **Source** remains set to **Deploy from a branch**, it may not work smoothly with the `deploy-pages` action, since that workflow is designed for an Actions-based deployment pipeline.

## Add a GitHub Actions Workflow for Pages Deployment

Create (or edit) the workflow file at:

* `.github/workflows/pages.yml`

This workflow checks out the repository, uploads the `frontend/` folder as the Pages artifact, and then deploys it using GitHub’s official Pages deployment action.

### Workflow Overview

* **Triggers**

  * Runs on pushes to `main`
  * Deploys only when files under `frontend/**` or the workflow file itself change (to avoid unnecessary deployments)
  * Also supports manual runs via `workflow_dispatch`

* **Permissions**

  * Grants the minimum required permissions for Pages deployment:

    * `contents: read`
    * `pages: write`
    * `id-token: write`

* **Deployment Behavior**

  * Uploads the `frontend/` directory directly as the site content
  * Deploys it to GitHub Pages using `actions/deploy-pages`

### Example `pages.yml`

```yaml
name: Deploy GitHub Pages (frontend)

on:
  push:
    branches: ["main"]
    # Deploy only when changes occur in frontend/ (skip condition)
    paths:
      - "frontend/**"
      - ".github/workflows/pages.yml"
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      # Upload frontend/ as-is as the Pages site artifact
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: frontend

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## Key Point: Upload the Folder You Want to Publish

The most important detail is the artifact upload step:

* `path: frontend`

This makes `frontend/` the root of your published Pages site. As long as `frontend/` contains the static files you want to serve (for example, an `index.html` at its top level), GitHub Pages will publish it exactly as provided.
