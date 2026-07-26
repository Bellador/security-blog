# security-blog

A GitHub Pages blog built on [Jekyll](https://jekyllrb.com/) + the
[Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme, set up for
DFIR / malware analysis / security write-ups.

## What's already done

- Chirpy theme wired in via the `jekyll-theme-chirpy` gem (see `Gemfile`)
- `_config.yml` pre-filled with placeholders (search for `CHANGE ME`)
- Comments configured for **giscus** (GitHub Discussions-based -- no
  third-party tracker, fits a security-focused blog)
- A sample post at `_posts/2026-07-25-linux-mongodb-compromise-investigation.md`
  showing the front matter / structure to reuse
- GitHub Actions workflow (`.github/workflows/pages-deploy.yml`) that builds
  and deploys automatically on push to `main`

## Setup

### 1. Create the GitHub repo

Push this to a new repo. If you want it served at
`https://<username>.github.io` directly (no subpath), name the repo
`<username>.github.io`. Otherwise name it anything (e.g. `security-blog`)
and it'll be served at `https://<username>.github.io/security-blog`.

```bash
cd security-blog
git init -b main
git add -A
git commit -m "Initial commit: Chirpy scaffold"
git remote add origin git@github.com:<username>/<repo>.git
git push -u origin main
```

### 2. Enable GitHub Pages

In the repo: **Settings -> Pages -> Build and deployment -> Source** ->
select **GitHub Actions**. The included workflow will then build and
deploy on every push to `main`.

### 3. Fill in `_config.yml`

Search the file for `CHANGE ME` and fill in:
- `url` -- your final Pages URL
- `github.username`
- `social.name`, `social.email` (consider an alias address rather than a
  personal one), `social.links`

### 4. Set up comments (optional)

If you want comments via giscus:
1. Enable **Discussions** on the repo (Settings -> General -> Features).
2. Go to [giscus.app](https://giscus.app), enter your repo, and it will
   generate a `repo_id` and `category_id`.
3. Paste those into the `comments.giscus` block in `_config.yml`.

If you'd rather not have comments at all, set `comments.provider:` to empty
in `_config.yml`.

### 5. Local preview (optional)

```bash
bundle install
bundle exec jekyll serve
```

Then open `http://127.0.0.1:4000`.

### 6. Write posts

Add files to `_posts/` named `YYYY-MM-DD-title.md` with front matter like:

```yaml
---
title: "Post Title"
date: 2026-07-25 09:00:00 +0200
categories: [DFIR, Linux]
tags: [linux, incident-response]
---
```

Chirpy supports GitHub-flavored Markdown, syntax-highlighted code blocks
(via Rouge), callout blocks (`{: .prompt-tip }`, `{: .prompt-warning }`,
`{: .prompt-danger }`), TOC generation, and image galleries -- see the
[Chirpy writing docs](https://chirpy.cotes.page/posts/write-a-new-post/) for
the full syntax reference.

## A note on operational hygiene for a security blog specifically

- Sanitize IOCs, hashes, and any real infrastructure details before
  publishing unless you intend them to be public.
- Static hosting (this setup) has no server-side attack surface of its own,
  but review anything you embed (iframes, scripts) before adding it.
- If you publish YARA rules, scripts, or samples alongside posts, consider a
  separate `tools/` or `assets/` subfolder so they're easy to audit and diff
  independently from post content.
