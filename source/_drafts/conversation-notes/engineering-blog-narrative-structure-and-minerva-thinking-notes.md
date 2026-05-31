---
title: Engineering Blog Narrative Structure and Minerva Thinking Notes
description: Notes about choosing blog-post structures, using Before-After-Why for engineering migration stories, and applying Minerva-style thinking habits as writing prompts.
tags:
- writing
- blog
- engineering-blog
- minerva
- thinking
---

## Question

- I wanted to know what structure is better for a blog post.
- The possible structure was compared with STAR.
- The topic example was a blog architecture change:
  - moving from Hexo to Astro Paper
  - separating content from site implementation
  - changing repository and workflow boundaries
- I also wanted to add:
  - a summary of Minerva University's thinking habits / HCs
  - an interview with Li Jiada, a Minerva graduate
  - the interview link:
    - https://www.youtube.com/watch?v=yMLMJuHHoIw&t=1403s

## Short Answer

- For blog posts, STAR is usually not the most natural structure.
- STAR feels like an interview answer or project report.
- Engineering blog posts usually work better with:
  - story
  - evolution
  - decision making
  - reasoning
  - tradeoffs

## Common Blog Post Structures

- Common engineering blog patterns:
  - Before -> After -> Why
  - Problem -> Solution -> Impact
  - Learning / Journey Narrative

- These work better than STAR when the post is about:
  - migration
  - workflow redesign
  - architecture evolution
  - personal infrastructure
  - tooling setup

## Why Not STAR?

- STAR means:
  - Situation
  - Task
  - Action
  - Result
- It is useful for interviews.
- But a blog post usually needs a different reader experience.
- Readers expect:
  - context
  - story flow
  - why the old approach stopped working
  - why the new approach was chosen
  - what tradeoffs came with the change

## Best Fit for This Topic

- The Hexo to Astro Paper content architecture topic fits:

```txt
Before -> After -> Why
```

- Reason:
  - the topic is about workflow and architecture evolution.
  - the reader needs to see how the old setup became insufficient.
  - the value comes from understanding the design choice.

## Before -> After -> Why Flow

- A natural flow:
  - how the blog worked before
  - what limitation appeared
  - why the limitation started to matter
  - what changed
  - what the new repository/workflow structure looks like
  - what tradeoffs and benefits came from the change

```txt
old workflow
    |
    v
new pressure / new needs
    |
    v
architecture decision
    |
    v
new workflow
    |
    v
why it is better
```

## Problem -> Solution -> Impact Flow

- This is also good for engineering writing.
- Example shape:
  - Problem:
    - content updates and frontend changes were mixed together.
  - Solution:
    - separate content and site implementation.
  - Impact:
    - clearer history.
    - easier framework upgrades.
    - more modular workflow.

- This is more direct than Before -> After -> Why.
- It is good when the post is short or practical.

## Learning / Journey Narrative Flow

- This is good when the post is personal.
- Example shape:
  - I started with Hexo.
  - I moved to Astro Paper.
  - I hit new customization needs.
  - I realized content and framework have different lifecycles.
  - I changed the repo architecture.
  - Here is what I learned.

- This style keeps the excitement of discovery.
- It works well for a personal dev blog.

## Suggested Opening Topic

- Possible title idea:

```txt
From Hexo to Astro: Separating Blog Content from Site Architecture
```

- Possible Chinese title idea:

```txt
從 Hexo 到 Astro：重新拆分 Blog 的 Content 與 Site Architecture
```

## Suggested Opening Draft

- Old workflow:
  - When I maintained the blog with Hexo, the workflow was simple.
  - Most of my attention was on writing posts.
  - The website design and structure stayed mostly fixed.

- New situation:
  - After moving to Astro Paper, the situation changed.
  - I started adjusting more than content.
  - I also changed:
    - layouts
    - components
    - styling
    - static pages
    - theme customization

- Key realization:
  - content and site implementation have different lifecycles.
  - blog posts evolve like knowledge records.
  - frontend framework code evolves like software.

## Problem With One Repository Boundary

- If content and framework stay too tightly coupled, several problems appear:
  - content updates and frontend changes mix together.
  - theme upgrades may affect post structure.
  - repository boundaries become unclear.
  - future migration cost becomes higher.
  - git history becomes harder to read.

## Architecture Decision

- The new architecture separates:
  - content management
  - site development

- Example direction:
  - content lives in a separate repository or separate source boundary.
  - the Astro site focuses on the static site implementation.
  - content can be introduced through a controlled path such as Git submodule or external source configuration.

## Why This Is Better

- Benefits:
  - content history becomes easier to track.
  - framework changes become easier to review.
  - Astro Paper can be upgraded more safely.
  - the migration path stays clearer.
  - site customization can evolve independently.

- The main idea:
  - content has a long-term knowledge lifecycle.
  - framework code has a technical lifecycle.
  - separating them makes both easier to maintain.

## Tradeoffs to Mention

- This architecture also has tradeoffs:
  - path configuration becomes more important.
  - build scripts may need extra setup.
  - Git submodules can add workflow complexity.
  - new contributors need to understand the repository structure.
  - deployment needs to know where content comes from.

- A good blog post should include these tradeoffs.
- That makes the decision feel grounded, not like pure preference.

## Good Blog Post Shape for This Story

- Recommended structure:

```txt
1. Old Hexo workflow
2. New Astro Paper customization needs
3. Problem: content and framework lifecycles diverged
4. Decision: isolate content from framework
5. New repository/workflow structure
6. Tradeoffs
7. Result and future migration benefit
```

## Minerva Thinking Habits / HCs

- Minerva University uses Habits of Mind and Foundational Concepts.
- They are often abbreviated as HCs.
- HCs are thinking tools that can transfer across contexts.
- They are useful as prompts for writing and reviewing an engineering blog post.

## About the Count

- Some older or secondary descriptions mention about 80 HCs.
- Minerva's current public intro material says there are more than 100 HCs and that the set changes over time.
- For this note, the important idea is not the exact number.
- The useful idea is:
  - build reusable thinking habits.
  - apply them across many domains.
  - use them to make decisions and explanations clearer.

## Minerva Four Core Competency Groups

- Minerva groups HCs under broad competencies.
- A useful writing-oriented summary:
  - thinking critically
  - thinking creatively
  - communicating effectively
  - interacting effectively

- These four groups can become a blog-post review checklist.

## Thinking Critically as Blog Review

- Use this group to check:
  - What claim am I making?
  - What evidence supports the claim?
  - What assumption am I making?
  - Is the problem real or only a preference?
  - What alternative explanations exist?

- For the Hexo-to-Astro story:
  - claim:
    - content and framework should be isolated.
  - evidence:
    - content edits and frontend changes have different lifecycles.
  - assumption:
    - future framework migration matters.

## Thinking Creatively as Blog Review

- Use this group to check:
  - What alternative designs did I consider?
  - Can I solve the problem with a smaller change?
  - Can I reuse an existing mechanism?
  - What new workflow becomes possible?

- For the blog architecture story:
  - alternatives may include:
    - keep everything in one repo.
    - use a subdirectory boundary.
    - use a Git submodule.
    - use a separate content repository.
    - use an external content path.

## Communicating Effectively as Blog Review

- Use this group to check:
  - Is the opening clear?
  - Does the reader know why the change matters?
  - Are terms like content, framework, source, and site defined?
  - Are diagrams or folder trees needed?
  - Is the conclusion memorable?

- For this story, a folder tree may help:

```txt
blog-root/
|-- astro-site/
|   `-- framework code
`-- content/
    `-- posts
```

## Interacting Effectively as Blog Review

- Use this group to check stakeholder impact.
- Possible stakeholders:
  - future me
  - readers
  - collaborators
  - deployment flow
  - search engines
  - framework maintainers

- Questions:
  - Who benefits from this architecture?
  - Who pays extra complexity?
  - What should be documented so the workflow remains usable?

## Minerva-Style Checklist for This Post

- Critical thinking:
  - Is the architecture problem clearly identified?
  - Are the assumptions visible?
- Creative thinking:
  - Are alternative structures mentioned?
  - Is the chosen design explained as a tradeoff?
- Communication:
  - Is the story readable from old workflow to new workflow?
  - Are examples concrete?
- Interaction:
  - Does the post explain how this helps future maintenance?
  - Does it mention complexity added by submodules or external content?

## Li Jiada Interview Link

- Related interview with Li Jiada, a Minerva graduate:
  - https://www.youtube.com/watch?v=yMLMJuHHoIw&t=1403s

- Why this belongs in the note:
  - it connects the writing structure discussion with Minerva-style learning habits.
  - it can be used as background inspiration for thinking tools.
  - it reminds me to write not only what changed, but how the reasoning changed.

## Possible Post Thesis

- A strong thesis:
  - This migration was not only a framework change.
  - It was a workflow boundary change.
  - The real insight was separating content lifecycle from framework lifecycle.

## Possible Final Paragraph

- The final paragraph can say:
  - Moving from Hexo to Astro Paper made me realize that a blog is both a knowledge base and a software project.
  - The content should be stable and portable.
  - The framework should be replaceable and customizable.
  - Separating them makes the system easier to reason about, upgrade, and migrate.

## References

- Minerva University learning model and HCs:
  - https://www.minerva.edu/pedagogy/
- Minerva introduction to Habits of Mind and Foundational Concepts:
  - https://www.minerva.edu/public/media/enrollment-center/Minerva-HCs-Intro.pdf
- Secondary article mentioning approximately 80 HCs:
  - https://ssir.org/articles/entry/creating_a_university_from_scratch
- Li Jiada Minerva graduate interview:
  - https://www.youtube.com/watch?v=yMLMJuHHoIw&t=1403s

## Final Summary

- For a blog post, use story-oriented structures instead of STAR.
- This topic fits `Before -> After -> Why` very well.
- The core story is content/framework lifecycle separation.
- Minerva-style HCs can be used as thinking prompts for reviewing the post.
- The exact HC count may depend on source/version, but the useful idea is reusable thinking tools.
