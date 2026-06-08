---
title: Astro Style Blog Survey
description: A short survey of Astro blog, documentation, and personal-site styles.
tags:
- astro
- blog
- survey
---

## AstroPaper family

- astro-paper
  - github: [satnaing/astro-paper](https://github.com/satnaing/astro-paper)
  - example site: https://astro-paper.pages.dev/

- astro-paper-s
  - github: [ziteh/astro-paper-s](https://github.com/ziteh/astro-paper-s)
  - example site: https://astro-paper-s.ziteh.dev/about
  - example site: https://blog.ziteh.dev/

- hoochanlon blog
  - github: [hoochanlon/hoochanlon.github.io](https://github.com/hoochanlon/hoochanlon.github.io)
  - example site: https://blog.hoochanlon.moe/

- web dong blog
  - github: [riceball-tw/web-dong-blog](https://github.com/riceball-tw/web-dong-blog)
  - example site: https://www.webdong.dev/zh-tw/post/slidev-build-presentation-with-markdown/

## Visual personal blog themes

- fuwari
  - github: [saicaca/fuwari](https://github.com/saicaca/fuwari)
  - example site: https://fuwari.vercel.app/posts/guide/#front-matter-of-posts

- twilight
  - github: [Spr-Aachen/Twilight](https://github.com/Spr-Aachen/Twilight/tree/main)
  - example site: https://twilight.spr-aachen.com/about/

## Landing page plus blog

- astrowind
  - github: [arthelokyo/astrowind](https://github.com/arthelokyo/astrowind)
  - example site: https://astrowind.vercel.app/useful-resources-to-create-websites

## Documentation style

- starlight
  - github: [withastro/starlight](https://github.com/withastro/starlight)
  - example site: https://starlight.astro.build/zh-cn/

- docusaurus
  - site: https://docusaurus.io/
  - note: Documentation-first static site generator; useful when the content model needs sidebars, docs pages, blog posts, and long-lived reference sections.
  - example site: https://wiwi.blog/
  - example note: Wiwi.Blog is built with Docusaurus, based on the site generator metadata and the `/use` page.

## Backup

- streamlit
  - site: https://streamlit.io/
  - note: Python-first app framework that can be used as a backup option for quickly publishing interactive tools or data-driven notes.
  - framework for past prototyping, such as learning site or data dashboard.

## Animation references

- math curve loaders
  - site: https://paidax01.github.io/math-curve-loaders/
  - github: [Paidax01/math-curve-loaders](https://github.com/Paidax01/math-curve-loaders)
  - note: Lightweight plain HTML/CSS/JavaScript gallery of mathematical curve based loading animations.
  - useful for: loading states, empty-state motion ideas, interactive visual accents, and code-copyable animation references.
  - details: Includes curve variants such as rose curves, Lissajous curves, hypotrochoids, cardioids, Cassini ovals, and Fourier-style paths, with formula notes and modal previews.

## Reference sites and articles

- pinchlime
  - site: https://pinchlime.com/about/
  - engine: Zola.
  - hosting/deploy: Netlify, according to the about page.
  - reference: https://www.owenyoung.com/
  - note: Current Pinchlime is a Zola-based personal blog with writing, newsletter backups, and snapshots. The about page says the Zola implementation and CSS were heavily inspired by Owen Young's blog and source code.

- pinchlime docusaurus snapshot
  - article: https://pinchlime.com/snapshots/why/why-do-i-want-to-build-another-website-by-docusaurus/
  - status: archived/outdated reference from the same website.
  - framework discussed: Docusaurus.
  - reference: https://brianlovin.com/
  - note: The article considered Docusaurus for a separate docs/list-style subsite, especially for a two-column docs layout inspired by Brian Lovin's `Stack` page. Do not treat this as the current main Pinchlime stack.

- wiwi blogroll
  - example site: https://wiwi.blog/blogroll
  - site: https://wiwi.blog/
  - engine: Docusaurus.
  - evidence: site generator metadata in the index page, and the `/use` page says the site is built with Docusaurus.
  - note: Useful reference for a Docusaurus-powered personal blog with blog, docs, `/use`, `/now`, and blogroll sections.

- rian astro choice article
  - example site: https://rian.cc/blog/explore-wordpress-hexo-vitepress-astro-choice

- learn harness engineering
  - example site: https://walkinglabs.github.io/learn-harness-engineering/zh-TW/
  - note: VitePress page that inspired this blog update.

## Simple compare

- Hexo
  - style: classic static blog.
  - content: Markdown-first.
  - strength: simple post workflow.
  - tradeoff: older theme ecosystem.
  - fit: stable personal notes.

- VitePress
  - style: documentation site.
  - content: Markdown-first.
  - strength: clean docs navigation.
  - tradeoff: blog features need extra work.
  - fit: learning notes and structured guides.

- Docusaurus
  - style: documentation-first site with blog support.
  - content: Markdown/MDX-first.
  - strength: sidebar docs, structured reference pages, blog, and SEO-oriented static output.
  - tradeoff: React/Node toolchain and docs-first assumptions.
  - fit: long-lived notes, `/docs`, `/use`, tool lists, and reference pages.

- Zola
  - style: fast static site generator.
  - content: Markdown-first.
  - strength: lightweight static blog with portable content.
  - tradeoff: more template/theme work when building docs-like layouts.
  - fit: independent personal blogs with simple static deployment.

- Astro
  - style: flexible content site.
  - content: Markdown and components.
  - strength: mix blog, docs, and custom pages.
  - tradeoff: more framework decisions.
  - fit: modern blog with isolated content.
