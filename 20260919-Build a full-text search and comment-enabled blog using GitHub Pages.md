---
title: "Build a full-text search and comment-enabled blog using GitHub Pages"
description: "If you want to implement a \"Markdown-based blog,\" \"full-text search,\" and \"comments\" on GitHub Pages,..."
pubDatetime: 2026-09-19T11:48:36.439Z
---

If you want to implement a "Markdown-based blog," "full-text search," and "comments" on GitHub Pages, the following configuration is easy to use.

## Recommended Configuration

```text
GitHub Repository
│
├─ src/content/blog/*.md / *.mdx
│        │
│        ▼
│   Astro
│   Static HTML generation
│        │
│        ▼
│   Pagefind
│   Full-text search index generation
│        │
│        ▼
├─ dist/
│   ├─ index.html
│   ├─ posts/...
│   └─ pagefind/...
│
│   GitHub Actions
│        │
│        ▼
└─ GitHub Pages
     │
     ├─ Article browsing
     ├─ Pagefind full-text search
     └─ giscus
          │
          ▼
       GitHub Discussions
```

GitHub Pages itself is a static hosting service that does not execute server-side processing in PHP/Python/Ruby, etc. On the other hand, you can build any static site generator with GitHub Actions and deploy it to Pages. ([GitHub Docs][1])

### First Choice

| Layer                  | Recommended Option |
| --------------------- | ------------------ |
| Content                 | Markdown / MDX     |
| Static Site Generator | **Astro**          |
| Full-Text Search        | **Pagefind**       |
| Comments                | **giscus**         |
| Comment Storage         | GitHub Discussions |
| CI/CD                 | GitHub Actions     |
| Hosting               | GitHub Pages       |
| Source Control          | GitHub             |

With this combination, you can operate without having a **search server, database, or application server.**

---

## 1. Static Site Generator

### When choosing Astro as the first choice

Place the blog posts in the following structure:

```text
src/content/blog/
├─ 2026-09-01-first-post.md
├─ 2026-09-10-github-pages.md
└─ 2026-09-19-pagefind.md
```

For example, the frontmatter might look like this:

```yaml
---
title: "Creating a Blog with GitHub Pages"
description: "Astro + Pagefind + giscus configuration"
date: 2026-09-19
tags:
  - GitHub
  - Astro
  - Pagefind
commentId: "2026-09-19-pagefind"
---
```

Astro provides official deployment procedures for GitHub Pages, allowing you to publish a pre-rendered site from GitHub Actions. ([Astro Docs][2])

### Comparison with other options

|                    | Astro | Hugo | Jekyll |
| ------------------ | ----- | ---- | ------ |
| Markdown blog        | ◎     | ◎    | ◎      |
| GitHub Pages       | ◎     | ◎    | ◎      |
| Pagefind integration | ◎     | ◎    | ○      |
| UI customization     | ◎     | ○    | ○      |
| Build speed          | ○     | ◎    | △      |
| Compatibility with JS/TS | ◎     | △    | △      |
| Proximity to GitHub Pages standard | ○     | ○    | ◎      |
| Future feature additions | ◎     | ○    | △      |

If you want a **simple** blog, Hugo is also a strong contender.

On the other hand, if you want to:

*   Create a sophisticated search UI
*   Use MDX
*   Use Web Components / React / Vue
*   Add features later

Astro is easier to configure.

Jekyll has high compatibility with GitHub Pages, but since you'll be adding post-processing like Pagefind, using a custom build with GitHub Actions is more convenient. GitHub also supports building and publishing with Actions for generators other than Jekyll. ([GitHub Docs][3])

---

# 2. Full-Text Search: Pagefind

**Pagefind is a good fit** for this.

The build flow is as follows:

```text
Markdown
   ↓
Astro build
   ↓
Static HTML
   ↓
Pagefind
   ↓
Static site with search index
```

For example, conceptually:

```bash
npm run build
npx pagefind --site dist
```

Pagefind parses the generated HTML and creates a search index, so no search API server is required.

### Japanese Support

This is important.

Pagefind explicitly supports Japanese (`ja`) and supports segmentation of non-space-separated text for Japanese, Chinese, and Korean. With `npx pagefind`, the extended release including this special language support is the default. ([Pagefind][4])

Therefore, you can include Japanese text in the search index:

```text
Implementing full-text search on GitHub Pages
```

---

## Make only the article body searchable

If you make the entire page searchable, the following will be indexed:

*   Navigation
*   Footer
*   Related articles
*   Sidebar

Therefore, it is a good idea to use the following in the article layout:

```html
<article data-pagefind-body>
  ...
</article>
```

Pagefind can limit the indexed area using `data-pagefind-body`. ([Pagefind][5])

This creates the following state:

```text
Header            ← Not included

Article title
Article body           ← Pagefind target
Code
Headings

Related articles           ← Not included
giscus comments     ← Not included
Footer             ← Not included
```

Since giscus comments are loaded on the browser after the build, they are not included in Pagefind's static index. This is more natural for a blog search.

---

## Tag filtering is also possible

Pagefind has a filtering mechanism. ([Pagefind][6])

Therefore, in the future, you can build a search UI like this:

```text
Search
┌───────────────────────────────┐
│ github pages                  │
└───────────────────────────────┘

Tags
☑ GitHub
□ Astro
□ Linux
□ Python

12 items
```

For example, the article would generate:

```html
<span data-pagefind-filter="tag">
  GitHub
</span>
```

---

# 3. Comments: giscus

**giscus** has very good compatibility with GitHub Pages.

The mechanism is as follows:

```text
Blog post
    │
    │ giscus iframe
    ▼
GitHub Discussions
    │
    ├─ Comments
    ├─ Replies
    └─ Reactions
```

You don't need to set up your own database. giscus stores comments in GitHub Discussions and displays comments and reactions on the blog. ([Giscus][7])

---

## You can also separate the comment repository

For example:

```text
myname/blog
    └─ Blog body

myname/blog-comments
    └─ GitHub Discussions
```

This is especially useful if you want:

```text
blog repository
    Private

blog-comments repository
    Public
```

giscus requires visitors to view the Discussion, so the destination repository must be **public**. In addition, you need to install the giscus App and enable Discussions. ([Giscus][7])

---

## Linking comments to articles

giscus can associate articles and Discussions based on:

*   `pathname`
*   URL
*   `title`
*   `og:title`
*   Specific string

([Giscus][7])

For a simple site, `pathname` is sufficient.

However, if you plan to use it for a long time, I recommend choosing a configuration that has an **article-specific ID**.

For example:

```yaml
commentId: "20260919-pagefind"
```

and:

```text
Article
/blog/pagefind/
    ↓
commentId
20260919-pagefind
    ↓
GitHub Discussion
```

This way, even if:

```text
/blog/pagefind/
    ↓ URL change
/articles/pagefind/
```

it will be easier to maintain the association of comments.

It is also not affected by changing the title.

---

# 4. giscus limitations

The biggest limitation is that:

> **People who want to comment also need a GitHub account.**

With giscus, visitors either post using GitHub OAuth or comment directly on GitHub Discussions. ([GitHub][8])

Therefore, it is suitable if the target audience is:

```text
Engineers
OSS users
GitHub users
```

On the other hand, if it is a general consumer blog and you want anonymous or semi-anonymous comments like:

```text
Name
Email
Body
[Submit]
```

giscus is not a good fit. In that case, you will need to use an external commenting service or a commenting API using Cloudflare Workers / Supabase.

---

# 5. GitHub Actions

Keep the deployment pipeline simple.

```text
git push
   ↓
GitHub Actions
   │
   ├─ npm install
   │
   ├─ Astro build
   │
   ├─ Pagefind index
   │
   └─ Pages artifact
   ↓
GitHub Pages
```

GitHub Pages now officially supports arbitrary static site builds using custom Actions workflows. ([GitHub Docs][1])

Therefore, you don't need to manage the `gh-pages` branch manually.

---

# 6. Repository configuration plan

Finally, it might look like this:

```text
blog/
├─ .github/
│  └─ workflows/
│     └─ deploy.yml
│
├─ src/
│  ├─ components/
│  │  ├─ Search.astro
│  │  ├─ Comments.astro
│  │  ├─ Header.astro
│  │  └─ Footer.astro
│  │
│  ├─ content/
│  │  └─ blog/
│  │     ├─ post-a.md
│  │     ├─ post-b.md
│  │     └─ post-c.md
│  │
│  ├─ layouts/
│  │  └─ BlogPost.astro
│  │
│  └─ pages/
│     ├─ index.astro
│     ├─ search.astro
│     └─ blog/
│
├─ public/
│  ├─ favicon.svg
│  └─ ...
│
├─ astro.config.mjs
├─ package.json
└─ tsconfig.json
```

After generation:

```text
dist/
├─ index.html
├─ search/
├─ blog/
└─ pagefind/
   ├─ pagefind.js
   ├─ pagefind-ui.js
   └─ Search index...
```

---

# 7. Search page

For example:

```text
/
├─ Article list
│
├─ /blog/xxx/
│      Article
│      ↓
│      giscus comments
│
└─ /search/
       ↓
   Pagefind UI
```

Place the following in the header:

```text
Blog        Tags        About        🔎 Search
```

If you put the search state in the URL like this:

```text
/search/?q=github
```

it will be easier to share links to search results.

---

# 8. Advantages of this configuration

It becomes easier to manage.

```text
                Managed
──────────────────────────
Web Server       None
Database         None
Search Server    None
Comment Server   None
CMS Server       None

GitHub Repo       Yes
GitHub Actions    Yes
GitHub Pages      Yes
GitHub DiscussionsYes
```

This means fewer points of failure.

Adding an article basically involves:

```text
Add Markdown
      ↓
git push
      ↓
Automatic build
      ↓
Search index update
      ↓
Publish
```

---

## It is a good idea to use this configuration as a basis.

```text
           GitHub
             │
      ┌──────┴──────┐
      │             │
 Blog Repository  Comments Repository
      │             │
 Markdown       Discussions
      │             ▲
    Astro           │
      │           giscus
   Pagefind          │
      │             │
      └──────┬──────┘
             │
       GitHub Pages
```

In particular, **Astro + Pagefind + giscus + GitHub Actions + GitHub Pages** is a very good fit for the requirements of "static blog, Japanese full-text search, and comments" this time.

If you move on to the next design, deciding on **① URL design, ② Markdown Frontmatter specification, ③ Pagefind search UI, ④ giscus Discussion mapping, ⑤ GitHub Actions `deploy.yml`** will put you in a state where you can start implementation directly.
