---
title: Using External Markdown Posts in astro-paper
pubDatetime: 2026-05-29T00:00:00+08:00
modDatetime: 2026-05-29T00:00:00+08:00
description: Notes on pointing astro-paper to Markdown posts outside the framework and adapting Hexo frontmatter.
tags:
- astro
- astro-paper
- markdown
- hexo
---

When I was using Hexo for blogging, my goal was to focus on the content.
I chose to maintain the Hexo framework and my post files in the same repository.
Now I plan to maintain the astro-paper framework and customize the blog style more deeply. I want to keep the content isolated from the blog framework, making migration and history tracking easier.

## Original Hexo Structure
Hexo manages the Markdown source under `source/_posts`.
It has a CMS (Content Management System) that keeps draft ideas under `source/_drafts`.
Although Astro has its own draft system, I will keep the Hexo folder structure for now.

## Steps to Use an External Content Directory in astro-paper
### Folder structure
astro-paper stores blog posts under `src/data/blog` by default.

The folder structure looks like this:

```txt
hsufit.github.io/
|-- astro-paper-hsufit/
|   |-- src/
|   |   |-- content.config.ts
|   |   `-- data/
|   |       `-- blog/
|   |           |-- adding-new-post.md
|   |           |-- how-to-configure-astropaper-theme.md
|   |           |-- how-to-integrate-giscus-comments.md
|   |           `-- ...
|   `-- ...
|
`-- source/
    |-- _posts/
    |   |-- hello-world.md
    |   |-- hexo-quick-start.md
    |   |-- git-useful-commands.md
    |   `-- ...
    |
    `-- _drafts/
        |-- astro-paper-after-start.md
        |-- astro-paper-quick-start.md
        `-- ...
```

### Update the target path in astro-paper
The key file is:

```txt
astro-paper-hsufit/src/content.config.ts
```

Original setting:

```ts
export const BLOG_PATH = "src/data/blog";
```

Updated setting:

```ts
// Use the markdown source out of the framework
// export const BLOG_PATH = "src/data/blog";
export const BLOG_PATH = "../source/_posts";
```

### Pitfalls
When I pointed astro-paper to the old Hexo posts, the content still did not show on the page.

The error looked like this:

```txt
[InvalidContentEntryDataError] blog -> find-device-by-mac data does not match collection schema.

  pubDatetime: pubDatetime: Required

  Hint:
    See https://docs.astro.build/en/guides/content-collections/ for more information on content schemas.
```

This means the folder path was already working. astro-paper found the Markdown file, but rejected it because the frontmatter did not match the blog collection schema.

The original Hexo posts do not have all the required fields in the astro-paper schema.

The missing required fields are:

- `pubDatetime`
- `description`

Hexo uses `date`, but astro-paper uses `pubDatetime`.

Another small trap is `tags`.

Hexo allows a string like this:

```yaml
tags: hexo
```

But astro-paper expects an array:

```yaml
tags:
- hexo
```

### Additional Information
Changing `BLOG_PATH` only tells astro-paper where to find the Markdown files.

After astro-paper finds the files, it still checks every post with the blog collection schema:

```ts
const blog = defineCollection({
  loader: glob({ pattern: "**/[^_]*.md", base: `./${BLOG_PATH}` }),
  schema: ({ image }) =>
    z.object({
      author: z.string().default(SITE.author),
      pubDatetime: z.date(),
      modDatetime: z.date().optional().nullable(),
      title: z.string(),
      featured: z.boolean().optional(),
      draft: z.boolean().optional(),
      tags: z.array(z.string()).default(["others"]),
      ogImage: image().or(z.string()).optional(),
      description: z.string(),
      canonicalURL: z.string().optional(),
      hideEditPost: z.boolean().optional(),
      timezone: z.string().optional(),
    }),
});
```

In Zod, a field without `.optional()`, `.nullable()`, or `.default(...)` is required.

That is why every post must provide:

- `pubDatetime`
- `description`
