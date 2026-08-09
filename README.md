# [Lee's Engineering Notes](https://lee259.github.io)

Personal blog about:

- **AI Agents** — Coding agents, memory systems, agent architecture
- **Frontend Infrastructure** — Scalable systems, tooling, component libraries
- **Developer Tools** — Productivity tools, automation, CLI utilities

## Tech Stack

- **Framework:** [Astro](https://astro.build)
- **Hosting:** GitHub Pages
- **Deployment:** GitHub Actions (auto-deploy on push to `main`)

## Development

```bash
npm install
npm run dev    # local dev server at http://localhost:4321
npm run build  # static build to dist/
npm run preview # preview the build
```

## Writing

Posts are markdown files in `src/content/posts/`. For bilingual posts, create two files with the same slug:

- `my-post.md` (English, default)
- `my-post-zh.md` (Chinese)

The dynamic route `[slug].astro` automatically groups them and provides a language switcher.

Post frontmatter:

```yaml
---
title: "Your Post Title"
description: "Brief description"
date: 2025-01-01
tags: ["Tag1", "Tag2"]
---
```

## License

MIT