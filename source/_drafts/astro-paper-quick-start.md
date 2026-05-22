---
title: Astro Paper Quick Start
description: Notes from moving Hexo content into AstroPaper while keeping content outside the framework.
tags:
- astro-paper
---

## Story

- I wanted a new AstroPaper shell.
- I also wanted old Hexo content untouched.
- Content should live outside the framework.
- The framework should be replaceable.
- The old URLs should still work.
- The homepage should still feel like the old blog.

## Install issue

- I created the AstroPaper project.
- I cloned the template repo.
- I ran `npm install`.
- I ran `npm run build`.
- The build failed in `astro.config.ts`.
- The error mentioned `fontProviders`.
- `astro/config` did not export it.
- The code expected a newer Astro.
- `package-lock.json` installed older Astro.
- `pnpm-lock.yaml` had newer packages.
- The lockfiles disagreed.
- AstroPaper updates fast.
- Vite and Astro versions move together.
- A stale lockfile breaks the template.

Design choice:

- Use one package manager.
- Prefer the template lockfile.
- Remove the stale lockfile if using npm.
- Reinstall dependencies cleanly.
- Check `astro` version after install.

Useful commands:

```powershell
npm ls astro
pnpm list astro
```

## External posts

- Hexo posts already lived in `source/_posts`.
- AstroPaper expected `src/data/blog`.
- I changed the collection path.
- `BLOG_PATH` points outside the framework.
- Astro can still load the Markdown.
- The first sync failed.
- Hexo frontmatter used `date`.
- AstroPaper needs `pubDatetime`.
- AstroPaper also needs `description`.
- Some Hexo tags were strings.
- AstroPaper expects tag arrays.

Design choice:

- Keep posts in `source/_posts`.
- Add AstroPaper metadata to each post.
- Keep old Hexo `date` for history.
- Add `pubDatetime` for AstroPaper.
- Add `description` for cards and SEO.
- Use array tags everywhere.

Important file:

```txt
astro-paper-hsufit/src/content.config.ts
```

Post example:

```yaml
date: 2020-07-14 21:55:01
pubDatetime: 2020-07-14T21:55:01+08:00
description: Find a LAN device by MAC address.
tags:
- network
- tool
```

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

Design choice:

- Generate static compatibility pages.
- Use `pubDatetime` for the date path.
- Reuse `getPath` for the new target.
- Keep the route independent from posts UI.

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
