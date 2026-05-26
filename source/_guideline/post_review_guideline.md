# Blog Review Guideline

## Review Priority

1. Grammar
2. Information correctness
3. Section smoothness

## Grammar

- Fix spelling first.
- Fix verb tense.
- Fix singular and plural forms.
- Keep sentences natural.
- Keep the author's excited style.
- Do not flatten vivid wording into boring formal prose.
- Preserve phrases with personality when they are understandable.

Examples:

- `Welcom` -> `Welcome`
- `is renew` -> `is renewed`
- `framkwork` -> `framework`
- `we finally no need` -> `I finally do not need`
- `try different style` -> `try a different style`

## Voice

- Keep energetic wording when it fits the post.
- Prefer light polish over heavy rewriting.
- Keep phrases like `LLM super power` if they express the intended mood.
- Make the grammar correct without removing the spark.

Example:

```md
After several years, this blog is renewed with LLM super power.
```

## Correctness

- Check technical claims against the repo when possible.
- Confirm package versions from `package.json`.
- Confirm config behavior from source files.
- Verify linked issues before summarizing them.
- Say what is known and what still needs local confirmation.

Examples:

- Syntax highlighting is configured in `astro.config.ts`.
- Code block styling is in `src/styles/typography.css`.
- Post copy buttons are added in `PostDetails.astro`.
- Astro 6 and Tailwind issues should be described as compatibility issues, not as a universal failure.

## Smoothness

- Keep the story order clear.
- Start with motivation.
- Then explain the framework choice.
- Then list goals.
- Then show commands.
- Put pitfalls after the happy path.
- Put environment details at the end.

Suggested order:

1. Why I renewed this blog
2. Framework choice
3. Nice-to-have design goals
4. Quick start
5. Pitfall
6. Environment

## Frontmatter

- Keep `title`, `pubDatetime`, `description`, and `tags`.
- Use `featured: false` for normal posts.
- Use `draft: false` for published posts.
- Use `modDatetime` only when it represents a meaningful update.
- Keep old Hexo `date` only if preserving migration history is useful.
