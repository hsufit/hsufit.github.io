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

## Google verification and sitemap

- I wanted to track the site in Google Search Console.
- My first try was on the old Hexo page.
- I added `google-site-verification` into the old Hexo `index.html`.
- That proved site ownership.
- But it did not mean Google would index the page.
- Later I found the page still was not indexed.
- The problem was discoverability.
- There was no clear incoming link from outside.
- The sitemap also had not been added.
- Google could verify the site.
- But Google still needed a way to discover the pages.
- Verification and indexing are different steps.
- Verification says: this site belongs to me.
- Sitemap says: these are the pages I want you to crawl.
- Links say: this page is connected to the web.

Design choice:

- Do the verification flow again in AstroPaper.
- Do not hardcode the verification meta tag in the page.
- Use AstroPaper's existing environment variable support.
- Set `PUBLIC_GOOGLE_SITE_VERIFICATION` at deploy stage.
- Let `Layout.astro` generate the meta tag.
- Keep the verification token outside content.
- Keep `SITE.website` pointed to the production URL.
- Use Astro's sitemap integration.
- Submit the generated sitemap in Google Search Console.

AstroPaper flow:

- `Layout.astro` already reads `PUBLIC_GOOGLE_SITE_VERIFICATION`.
- If the env var exists, it adds:

```html
<meta name="google-site-verification" content="..." />
```

- `astro.config.ts` already uses `@astrojs/sitemap`.
- `SITE.website` decides the final sitemap URLs.
- Running build generates the sitemap files.
- The expected sitemap entry is:

```txt
https://hsufit.github.io/sitemap-index.xml
```

How to check:

- Build the AstroPaper site.
- Open the generated homepage HTML.
- Confirm the Google verification meta tag exists.
- Open `dist/sitemap-index.xml`.
- Confirm the post URLs are listed through the sitemap.
- Open `dist/robots.txt`.
- Confirm it points to the sitemap.
- Deploy the site.
- Submit `https://hsufit.github.io/sitemap-index.xml` to Google Search Console.
- Use URL Inspection for important pages.

Important files:

```txt
astro-paper-hsufit/src/layouts/Layout.astro
astro-paper-hsufit/src/config.ts
astro-paper-hsufit/astro.config.ts
astro-paper-hsufit/src/pages/robots.txt.ts
```

## GitHub Actions deploy flow

- I wanted the AstroPaper site to deploy automatically.
- The source branch is `astro-paper`.
- The output branch is `gh-pages`.
- The framework lives in a submodule:

```txt
astro-paper-hsufit
```

- The content lives in the root repo:

```txt
source/_posts
```

- The CI flow needs both pieces.
- GitHub Actions must checkout submodules.
- The build must run inside `astro-paper-hsufit`.
- The deploy step must publish `astro-paper-hsufit/dist`.

Final flow:

- Checkout the source branch.
- Checkout submodules recursively.
- Setup Node.js.
- Install dependencies with `npm ci`.
- Build the AstroPaper site.
- Pass Google Search Console verification by env var.
- Add `.nojekyll`.
- Deploy `dist` to `gh-pages`.

Workflow shape:

```yaml
on:
  push:
    branches:
      - astro-paper
  workflow_dispatch:

permissions:
  contents: write
```

- `workflow_dispatch` lets me run deploy manually.
- `contents: write` lets the action push to `gh-pages`.
- The deploy action uses `GITHUB_TOKEN`.

Submodule choice:

```yaml
- name: Checkout source
  uses: actions/checkout@v5
  with:
    submodules: recursive
    fetch-depth: 0
```

- `submodules: recursive` is required.
- Without it, `astro-paper-hsufit` may be empty or incomplete.
- `fetch-depth: 0` keeps full history available.

Install and build:

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v5
  with:
    node-version: 22
    package-manager-cache: false

- name: Install dependencies
  working-directory: astro-paper-hsufit
  run: npm ci

- name: Build site
  working-directory: astro-paper-hsufit
  env:
    PUBLIC_GOOGLE_SITE_VERIFICATION: ${{ vars.PUBLIC_GOOGLE_SITE_VERIFICATION }}
  run: npm run build
```

- `working-directory` is important.
- The Astro app is not at the repo root.
- `npm ci` is better for CI than `npm install`.
- It requires a committed `package-lock.json`.
- The Google verification value comes from GitHub Actions variables.
- At first, I only looked at the workflow YAML.
- The YAML line was correct.
- But the GitHub repository variable also had to exist.
- The workflow reads from `vars.PUBLIC_GOOGLE_SITE_VERIFICATION`.
- That means the value must be configured as a repository variable.
- If I put the value in GitHub Secrets instead, the YAML must use `secrets`.

Missing setting lesson:

- Repository variable is the stored value in GitHub.
- Environment variable is the runtime value passed to the build command.
- The workflow bridges them:

```yaml
env:
  PUBLIC_GOOGLE_SITE_VERIFICATION: ${{ vars.PUBLIC_GOOGLE_SITE_VERIFICATION }}
```

- This means:
  - read `PUBLIC_GOOGLE_SITE_VERIFICATION` from GitHub repository variables
  - expose it as `PUBLIC_GOOGLE_SITE_VERIFICATION` during `npm run build`
  - let Astro read it through `astro:env/client`
- The name must match in all three places.
- GitHub variable name:

```txt
PUBLIC_GOOGLE_SITE_VERIFICATION
```

- Workflow env name:

```txt
PUBLIC_GOOGLE_SITE_VERIFICATION
```

- Astro env schema name:

```txt
PUBLIC_GOOGLE_SITE_VERIFICATION
```

Deploy:

```yaml
- name: Disable Jekyll
  run: touch astro-paper-hsufit/dist/.nojekyll

- name: Deploy to gh-pages
  uses: peaceiris/actions-gh-pages@v4.1.0
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_branch: gh-pages
    publish_dir: astro-paper-hsufit/dist
```

- `.nojekyll` is needed for GitHub Pages static assets.
- It prevents GitHub Pages from treating underscore folders specially.
- `publish_dir` must point to Astro's build output.

Small future work:

- GitHub also has a newer official Pages deploy flow.
- It uses `actions/upload-pages-artifact` and `actions/deploy-pages`.
- That flow deploys through the GitHub Pages environment.
- The current workflow uses `peaceiris/actions-gh-pages`.
- The main replacement would be the final deploy stage.
- The build stage can stay almost the same.
- This is a cleanup task for later, not required for the first working deploy.

CI issue: npm cache path

- The first CI issue was from dependency caching.
- The error said:

```txt
Some specified paths were not resolved, unable to cache dependencies.
```

- The workflow had `setup-node` cache settings.
- The cache path was fragile for this nested project.
- I removed the explicit cache setting first.
- Later, after moving to `setup-node@v5`, I disabled automatic package-manager cache.

Design choice:

- Keep dependency install simple.
- Do not make caching part of the first working deploy.
- Add cache later only after the deploy is stable.

CI issue: missing lockfile

- `npm ci` failed when CI could not find a valid `package-lock.json`.
- `npm ci` requires a lockfile.
- The fix was to generate and commit `astro-paper-hsufit/package-lock.json`.
- The preferred flow is:

```powershell
cd astro-paper-hsufit
npm install
```

- Then commit the lockfile.
- Keep CI using `npm ci`.

Design choice:

- Do not replace `npm ci` with `npm install` in CI.
- Use `npm install` locally to update the lockfile.
- Use `npm ci` in GitHub Actions for reproducible deploy builds.

CI issue: Rollup optional dependency

- Another failure was:

```txt
Cannot find module '@rollup/rollup-linux-x64-gnu'
```

- This is related to npm optional dependencies.
- Rollup uses platform-specific native packages.
- The Linux CI runner needs the Linux Rollup optional package.
- The clean fix is to refresh the lockfile locally.
- Then commit the updated `package-lock.json`.

Design choice:

- Do not delete `package-lock.json` inside CI.
- Do not run `npm install` inside CI as a workaround.
- Fix the lockfile at the source.
- Let `npm ci` install exactly what the lockfile describes.

CI warning: Node 20 actions

- GitHub Actions warned that Node.js 20 actions are deprecated.
- `actions/checkout@v4` and `actions/setup-node@v4` used Node 20.
- I updated them to:

```yaml
actions/checkout@v5
actions/setup-node@v5
```

- `peaceiris/actions-gh-pages@v4` also used Node 20.
- I pinned it to:

```yaml
peaceiris/actions-gh-pages@v4.1.0
```

- That version runs on Node 24.
- This is different from `node-version: 22`.
- `node-version: 22` controls the Node.js used to build Astro.
- The action runtime controls how GitHub runs the action itself.

Important GitHub settings:

- Set Pages source:

```txt
Settings -> Pages -> Deploy from a branch -> gh-pages / root
```

- Set the verification variable:

```txt
Settings -> Secrets and variables -> Actions -> Variables
PUBLIC_GOOGLE_SITE_VERIFICATION=...
```

Submodule reminder:

- Changes inside `astro-paper-hsufit` belong to the submodule repo.
- The root repo only records the submodule pointer.
- If I update AstroPaper code:
  - push the submodule repo first
  - update the submodule pointer in the root repo
  - push the root repo

Important files:

```txt
.github/workflows/deploy.yml
astro-paper-hsufit/package.json
astro-paper-hsufit/package-lock.json
astro-paper-hsufit/astro.config.ts
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
