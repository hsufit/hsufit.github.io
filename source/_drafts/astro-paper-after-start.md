---
title: Astro Paper Quick Start
description: Notes from moving Hexo content into AstroPaper while keeping content outside the framework.
tags:
- astro
- astro-paper
---

## Story

- I wanted a new AstroPaper shell.
- I also wanted old Hexo content untouched.
- Content should live outside the framework.
- The framework should be replaceable.
- The old URLs should still work.
- The homepage should still feel like the old blog.

## Old Hexo redirects

- Old Hexo URLs had dates.
- New AstroPaper URLs use `/posts`.
- Old URL example:

```txt
/2020-07-14/find-device-by-mac/
```

- New URL example:

```txt
/posts/find-device-by-mac/
```

- GitHub Pages cannot set server redirects.
- Static redirect pages are needed.
- Astro can generate them.
- A catch-all route matches old paths.
- `getStaticPaths` creates one page per post.
- The page uses meta refresh.
- The page also uses canonical.
- JavaScript redirects users faster.
- `noindex` keeps old pages out of search.
- I checked the old URL in DevTools.
- The request returned `200 OK`.
- That was expected.
- The page is a static HTML file.
- GitHub Pages served it normally.
- The redirect happened inside the browser.
- It was not an HTTP `301`.
- Meta refresh is weaker for SEO.
- Canonical helps search engines consolidate.
- `noindex` avoids indexing the old page.
- A real `301` would be better.
- GitHub Pages cannot configure it directly.

Design choice:

- Generate static compatibility pages.
- Use `pubDatetime` for the date path.
- Reuse `getPath` for the new target.
- Keep the route independent from posts UI.
- Accept `200 OK` for static-host compatibility.
- Use meta refresh as the SEO fallback.
- Use JavaScript for faster user navigation.
- Do not expect Network panel to show `301`.

How to check:

- Open an old Hexo URL.
- Confirm it loads a generated HTML page.
- Confirm the page contains meta refresh.
- Confirm the page contains canonical.
- Confirm the final URL becomes `/posts/.../`.
- Confirm the old page has `noindex`.
- Check page source, not only Network.

Delay choice:

- Use `0` seconds for old post redirects.
- A delay slows users down.
- A delay does not turn it into `301`.
- A delay can make crawlers treat it less clearly.
- Add delay only for a human-facing notice page.
- Keep instant redirect for migrated permalinks.

Important files:

```txt
astro-paper-hsufit/src/pages/[...hexoSlug].astro
astro-paper-hsufit/src/utils/hexoRedirect.ts
```

## External about page

- Hexo had `source/about/index.md`.
- AstroPaper had `src/pages/about.md`.
- That mixed content with framework.
- I replaced the Markdown route.
- `about.astro` imports external Markdown.
- The layout stays in AstroPaper.
- The content stays in `source`.

Design choice:

- Keep page content outside the framework.
- Keep layout inside the framework.
- Use an Astro wrapper route.
- Import Markdown as a component.
- Import frontmatter for the title.

Important files:

```txt
source/about/index.md
astro-paper-hsufit/src/pages/about.astro
astro-paper-hsufit/src/layouts/AboutLayout.astro
```

## Squirrel page

- Hexo had a custom `squirrel` page.
- AstroPaper did not know that route.
- I added `squirrel.astro`.
- It imports `source/squirrel/index.md`.
- I added a `SquirrelLayout`.
- The header links to `/squirrel`.

Design choice:

- Treat custom pages like About.
- Use one wrapper per external page.
- Keep each layout explicit.
- Avoid putting special pages in blog posts.

Important files:

```txt
source/squirrel/index.md
astro-paper-hsufit/src/pages/squirrel.astro
astro-paper-hsufit/src/layouts/SquirrelLayout.astro
astro-paper-hsufit/src/components/Header.astro
```

## Nut icon

- I wanted the Squirrel page in the header.
- Text worked first.
- Then I wanted an icon.
- I tried a squirrel SVG.
- The original SVG was huge.
- It came from a filled trace.
- It had a very large viewBox.
- It did not match `IconArchive`.
- I generated a nut icon idea first.
- I used Gemini with Nano Banana.
- I asked for a line-style nut.
- The output was a PNG.
- I converted PNG to SVG.
- I used Potrace for tracing.
- Potrace made a large SVG.
- The SVG path was usable.
- The wrapper was not icon-ready.
- The header icons expect `24x24`.
- I resized the SVG wrapper.
- I used `viewBox="0 0 24 24"`.
- I used `width="24"`.
- I used `height="24"`.
- The icon then fit the header.
- The next issue was color.
- `IconArchive` is a stroke icon.
- It uses `stroke="currentColor"`.
- The active class used `stroke-accent`.
- The nut icon was a filled shape.
- It used `fill="currentColor"`.
- `stroke-accent` did not change its visible color.
- I tried making it stroke-based.
- A filled trace is not a true outline icon.
- With `fill="none"`, it can look weak.
- With `fill="currentColor"`, it colors like text.
- I used an LLM to shrink the wrapper.
- The LLM kept the path data.
- The LLM changed the SVG contract.
- The final pass was design work.
- I compared it with `IconArchive`.
- I adjusted size and color behavior.
- I tested it in the header.

Design choice:

- Match size inside the SVG file.
- Keep one-off layout classes in `Header.astro`.
- Use stroke icons with `stroke-accent`.
- Use filled icons with `text-accent` or `fill-accent`.
- Prefer a real outline SVG if matching Tabler icons.
- Avoid converting a filled trace into an outline by only changing `fill`.
- Remove unused icon imports.

Workflow:

- Generate line-style nut PNG.
- Use Gemini with Nano Banana.
- Convert PNG to SVG.
- Use [Potrace](https://potrace.sourceforge.net/?utm_source=chatgpt.com). 
- Keep the traced path.
- Replace the huge SVG wrapper.
- Use a `24x24` viewBox.
- Use `currentColor`.
- Choose `fill` or `stroke` intentionally.
- Add header active-state styling.
- Compare with `IconArchive`.
- Finish by checking the page.

Important files:

```txt
astro-paper-hsufit/src/assets/icons/IconArchive.svg
astro-paper-hsufit/src/assets/icons/IconNut.svg
astro-paper-hsufit/src/assets/icons/IconNut2.svg
astro-paper-hsufit/src/components/Header.astro
```

## Banner

- The AstroPaper homepage felt generic.
- The old Hexo blog had a banner.
- The old title was `Do. Or do not.`
- The old subtitle was `There is no try.`
- The old image was `banner.jpg`.
- The image existed in the `hexo` branch.
- I copied it into `public/images`.
- I replaced the homepage intro.
- The post list stayed unchanged.

Design choice:

- Use a real image asset.
- Keep the banner full-width.
- Keep posts in the normal app layout.
- Preserve the old blog identity.
- Avoid changing shared layout components.

Important files:

```txt
astro-paper-hsufit/src/pages/index.astro
astro-paper-hsufit/public/images/banner.jpg
```

## Final shape

- `source` owns content.
- `astro-paper-hsufit` owns rendering.
- Posts come from `source/_posts`.
- About comes from `source/about`.
- Squirrel comes from `source/squirrel`.
- Redirects protect old Hexo links.
- The homepage keeps the old banner mood.


## Find an icon base from research of slidev
2026/05/26
- searching and learning to use slidev for vibe slide generation
- found a blog describe the use case and the free icon base for slidev, and it also useful for blog
pages:
- https://iconify.design/
- https://pictogrammers.com/library/mdi/
- https://icones.js.org/

reference:
- https://www.webdong.dev/zh-tw/post/slidev-build-presentation-with-markdown/
- https://sli.dev/features/icons