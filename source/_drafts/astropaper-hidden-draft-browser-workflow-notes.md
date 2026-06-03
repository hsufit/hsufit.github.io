---
title: AstroPaper Hidden Draft Browser Workflow Notes
description: Notes about adding a hidden draft browser in AstroPaper, keeping drafts separate from public posts, and excluding draft pages from Google, sitemap, RSS, and public navigation.
tags:
- astropaper
- draft
- astro
- seo
- github-pages
---

## Question

- I wanted a hidden draft page in AstroPaper.
- The goal was to browse notes under `source/_drafts` directly from the generated site.
- I also wanted these draft Markdown files to render even when they do not have the normal AstroPaper blog YAML fields.
- The normal blog schema requires fields such as:
  - `pubDatetime`
  - `description`
  - `tags`
- Draft notes are often rough notes, so requiring full post metadata would make the draft workflow too heavy.

## Main Idea

- Keep public posts and draft notes as two different content flows.
- Public posts still use the existing `blog` collection.
- Draft notes use a new relaxed `drafts` collection.
- This prevents draft notes from affecting:
  - homepage
  - post list
  - tags
  - archives
  - RSS
  - sitemap
  - normal public search/indexing flow

## Why A Separate Draft Collection

- The existing `blog` collection is designed for published posts.
- It validates required metadata to make public pages consistent.
- Draft notes have a different purpose:
  - quick research notes
  - rough converted conversations
  - incomplete article ideas
  - temporary technical records
- A separate `drafts` collection can use a relaxed schema:
  - `title` optional
  - `description` optional
  - `pubDatetime` optional
  - `tags` optional or defaulted
- This lets missing YAML fields stay valid for draft rendering.

## Hidden Draft Route

- The hidden route is:

```txt
/draft/
```

- The index page recursively lists Markdown files under:

```txt
source/_drafts
```

- Nested folders such as `conversation-notes` are preserved in the browser.
- Each draft file gets a detail page:

```txt
/draft/<folder>/<filename>/
```

- The draft title fallback rule is:
  - use frontmatter `title` if present
  - otherwise derive a readable title from the filename

## Why Not Use The Normal Blog Post Layout Directly

- AstroPaper post detail components expect full public post metadata.
- Draft notes may not have that metadata.
- The draft page should be simpler:
  - title
  - optional date
  - optional description
  - optional tags
  - source file path
  - rendered Markdown body
- This keeps draft rendering note-like instead of forcing it to be a polished article.

## Keeping Drafts Away From Google

- The draft pages are not meant to appear on Google.
- Several layers are used to reduce discoverability.

### No Navigation Link

- `/draft/` is not added to the header or public navigation.
- A normal visitor will not see a link to it from the site UI.

### Noindex Meta Tag

- Draft pages include:

```html
<meta name="robots" content="noindex, nofollow" />
```

- This tells search engines:
  - do not index this page
  - do not follow links from this page

### Sitemap Exclusion

- `/draft/` pages are filtered out from the sitemap.
- This is important because a sitemap is one of the main ways Google discovers pages.
- If draft pages are in the sitemap, Google can discover them even if the header does not link to them.

### Robots.txt Disallow

- `robots.txt` also includes:

```txt
Disallow: /draft/
```

- This gives crawlers another signal that the draft area should not be crawled.

## Why This Is Not Real Security

- This is hidden access, not authentication.
- GitHub Pages is static hosting.
- There is no server-side password check.
- Anyone who knows the exact `/draft/` URL can still open it.
- This workflow is useful for:
  - low-profile draft browsing
  - personal preview
  - keeping notes out of Google
  - avoiding accidental public navigation
- It is not suitable for:
  - private documents
  - secrets
  - credentials
  - unpublished content that must be protected

## Implementation Parts

- Content collection:
  - add a `drafts` collection in `src/content.config.ts`
  - point it to `../source/_drafts`
  - use optional/defaulted fields

- Draft index route:
  - create `src/pages/draft/index.astro`
  - recursively build a tree/list from all draft entries

- Draft detail route:
  - create `src/pages/draft/[...slug].astro`
  - render each Markdown draft with fallback metadata

- Shared layout:
  - allow a `robots` prop
  - use `robots="noindex, nofollow"` for draft pages
  - avoid assuming every page has `pubDatetime`

- Sitemap:
  - filter out paths starting with `/draft/`

- Robots:
  - add `Disallow: /draft/`

## Final Workflow

- Write rough notes under:

```txt
source/_drafts/
```

- Build the Astro site.
- Open:

```txt
https://hsufit.github.io/draft/
```

- Browse drafts from the generated tree.
- Public blog pages still come only from:

```txt
source/_posts/
```

## Key Lesson

- Draft browsing should be separated from public publishing.
- The public blog schema should stay strict.
- The draft schema can be relaxed.
- Google exclusion needs multiple layers:
  - no nav link
  - `noindex`
  - sitemap exclusion
  - `robots.txt`
- The result is convenient for personal draft preview, but it should not be treated as real private access.
