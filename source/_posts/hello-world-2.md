---
title: Hello World 2
date: 2026-05-26 00:00:00
pubDatetime: 2026-05-26T00:00:00+08:00
modDatetime: 2026-05-26T00:00:00+08:00
description: Notes about renewing this blog with AstroPaper while keeping content separate from the framework.
featured: false
draft: false
tags:
- astro
- astro-paper
---
Aha! Welcome to a new world built with [Astro](https://astro.build/).
After several years, this blog is renewed with LLM super power.

It feels like a new era: I finally do not need to spend all my energy debugging framework issues, and can enjoy content creation again.

To explore the edge of knowledge and try a different style of blogging, I chose a different framework that:
- Is designed for Markdown-based blogging
- Supports redirecting old Hexo links
- Generates static pages for GitHub Pages
- Can be cloned and is ready to go

Nice to have:
- Content should live outside the framework, so Git history stays clean and framework upgrades stay easy
- The framework should be replaceable
- Auto deploy to Github Pages

## Quick Start
### Download the astro-paper<link to astro-paper github>
```bash
git clone https://github.com/satnaing/astro-paper
```
### Dev mode
```bash
npm run dev
```
### Build and preview
```bash
npm run build
npm run preview
```
### Deploy
Put the content in dist into your target folder.


### Pitfall
Vite and Tailwind CSS have a compatibility [issue](https://github.com/withastro/astro/issues/16542) that can cause Astro 6 builds to fail.
Avoid upgrading blindly to the latest version while the upstream ecosystem is still moving to Astro 6.


### Additional info
This quick start is based on:
node: v22.22.3
astro-paper: f3005328e548805226aba54414122c7174645e83

