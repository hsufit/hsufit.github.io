---
title: Google Search Console Indexing and AstroPaper Sitemap Notes
description: Notes about Google Search Console verification, why a verified page may still be unknown to Google, and how to apply sitemap and verification flow in AstroPaper.
tags:
- google-search-console
- seo
- sitemap
- astropaper
- github-pages
---

## Question

- I wanted to track my blog activity with Google Search Console.
- I first tried the flow on the old Hexo site.
- I added a Google verification meta tag into the old Hexo `index.html`.
- The verification step worked.
- But the page still was not indexed by Google.
- I wanted to know why verification did not lead to indexing.
- I also wanted to apply the same flow correctly in AstroPaper.

## Key Lesson

- Google Search Console verification and Google indexing are different things.
- Verification only proves that I can control the site.
- Indexing means Google has discovered, crawled, and decided to include the page.
- A site can be verified but still unknown to Google.

```html
<meta name="google-site-verification" content="xxxxx" />
```

- This tag only proves site ownership.
- It does not submit pages to Google.
- It does not generate a sitemap.
- It does not create external links.
- It does not force indexing.

## URL Inspection Flow

- Open Google Search Console.
- Select the site property.
- Use the URL Inspection tool.
- Enter the homepage URL:

```txt
https://hsufit.github.io/
```

- This tool answers the important question:
  - Is the URL known to Google?
  - Has Google crawled it?
  - Is indexing allowed?
  - Was a sitemap detected?
  - Was a referring page detected?

## Possible Results

- If the result says `URL is on Google`, the page is indexed.
- If `site:` search does not show it yet, the public search result may not have updated.
- If the result says `URL is not on Google`, the page is not indexed.
- The next step is to read the Page indexing reason.

## Common Indexing Reasons

- `Discovered - currently not indexed`
  - Google knows the URL.
  - Google has not crawled it yet.
  - This is common for new or low-authority sites.
  - The next step is to request indexing.

- `Crawled - currently not indexed`
  - Google fetched the page.
  - Google decided not to index it yet.
  - Possible causes:
    - thin content
    - duplicate content
    - weak internal links
    - unclear SEO structure

- `Blocked by robots.txt`
  - Google is not allowed to crawl the page.
  - Check:

```txt
https://hsufit.github.io/robots.txt
```

- `noindex`
  - The page contains a `noindex` directive.
  - Google will not index it.
  - Check the page source for:

```html
<meta name="robots" content="noindex" />
```

- `Duplicate, Google chose different canonical`
  - Google thinks another URL is the main version.
  - This is usually a canonical or duplicate-content issue.

## My Search Console Result

- The important message was:

```txt
URL is not on Google
Page is not indexed: URL is unknown to Google
```

- The Discovery section also showed:

```txt
Sitemaps: No referring sitemaps detected
Referring page: None detected
```

- The Crawl section showed:

```txt
Last crawl: N/A
Crawled as: N/A
Crawl allowed?: N/A
Page fetch: N/A
Indexing allowed?: N/A
```

- The canonical section showed:

```txt
User-declared canonical: N/A
Google-selected canonical: N/A
```

- This means Google had not reached the page yet.
- It was not a robots problem yet.
- It was not a canonical problem yet.
- It was not a content-quality problem yet.
- The page was simply unknown to Google.

## What Unknown to Google Means

- Google has no discovery path to the URL.
- No sitemap pointed to it.
- No referring page pointed to it.
- Googlebot had not crawled it.
- The page was outside Google's known link graph.

## Why the Old Hexo Attempt Was Not Enough

- I added the verification meta tag to the old Hexo `index.html`.
- That confirmed ownership.
- But the old flow did not solve discovery.
- The sitemap had not been submitted.
- There was no detected referring page.
- There was no clear external link path.
- Google could verify the site but still not know the page.

## What to Do First

- Submit a sitemap in Google Search Console.
- Make sure the sitemap can be opened in a browser.
- Make sure the homepage links to the posts.
- Make sure important pages are not orphan pages.
- Use URL Inspection.
- Click `Request indexing` for important pages.

## Sitemap Flow in AstroPaper

- AstroPaper already uses Astro's sitemap integration.
- The sitemap is configured in:

```txt
astro-paper-hsufit/astro.config.ts
```

- The production URL comes from:

```txt
astro-paper-hsufit/src/config.ts
```

- The important value is:

```ts
SITE.website
```

- For this blog, it should point to:

```txt
https://hsufit.github.io/
```

- Running the Astro build generates the sitemap.

```powershell
cd astro-paper-hsufit
npm.cmd run build
```

- The expected sitemap URL after deploy is:

```txt
https://hsufit.github.io/sitemap-index.xml
```

- This is the URL to submit in Google Search Console.

## robots.txt Flow

- AstroPaper also generates `robots.txt`.
- It should point search engines to the sitemap.
- The generated file should be available at:

```txt
https://hsufit.github.io/robots.txt
```

- It should contain a sitemap line similar to:

```txt
Sitemap: https://hsufit.github.io/sitemap-index.xml
```

## Google Verification Token

- The `google-site-verification` content value is not a secret.
- It is not a password.
- It is not an API key.
- It cannot be used to log in to Google Search Console.
- It only proves that I can modify the site HTML.
- If someone can modify my site, they already have the real control problem.
- The token itself is safe to appear in the public HTML.

## Why AstroPaper Uses an Environment Variable

- AstroPaper uses:

```txt
PUBLIC_GOOGLE_SITE_VERIFICATION
```

- This is not mainly for security.
- It is mainly for deployment configuration.
- The value can be different between environments:
  - local development
  - preview
  - production

- Benefits:
  - avoid hardcoding deployment-specific values
  - avoid changing source code for each site
  - keep templates easier to reuse
  - allow CI/CD systems to inject the value
  - avoid config conflicts in team projects

- The token is public.
- But using an env var still keeps the code cleaner.

## AstroPaper Verification Flow

- Set the env var during deployment:

```txt
PUBLIC_GOOGLE_SITE_VERIFICATION=xxxxx
```

- `Layout.astro` reads this value.
- If the env var exists, it emits:

```html
<meta name="google-site-verification" content="xxxxx" />
```

- I do not need to hardcode this in `index.astro`.
- The shared layout adds it to the page head.
- This keeps the verification concern outside the content files.

## Practical Checklist

- Set `SITE.website` to the production URL.
- Set `PUBLIC_GOOGLE_SITE_VERIFICATION` in the deploy environment.
- Run the Astro build.
- Confirm the homepage HTML contains the verification meta tag.
- Confirm `dist/sitemap-index.xml` exists.
- Confirm `dist/robots.txt` points to the sitemap.
- Deploy the site.
- Open the deployed sitemap URL.
- Submit the sitemap in Google Search Console.
- Inspect the homepage URL.
- Click `Request indexing`.
- Add real links from GitHub README or other external pages.

## Final Summary

- Verification tells Google that I own the site.
- Sitemap tells Google what pages exist.
- Links help Google discover and trust pages.
- My old Hexo attempt solved only verification.
- The AstroPaper flow should include both verification and sitemap submission.
- If Search Console says `URL is unknown to Google`, the first problem is discovery.
- The first fix is sitemap submission plus links, not content rewriting.

## References

- [Google site verification meta tag documentation](https://support.google.com/webmasters/answer/9008080#meta_tag_verification)
