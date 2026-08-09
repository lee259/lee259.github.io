# Lee's Engineering Notes

A collection of thoughts and practices about:

- **AI Agents** — Coding agents, memory systems, agent architecture
- **Frontend Infrastructure** — Scalable systems, tooling, component libraries
- **Developer Tools** — Productivity tools, automation, CLI utilities
- **Open Source** — [Agent Brain](https://github.com/lee259/agent-brain) and other projects

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

Add a new `.astro` file in `src/pages/posts/` and import the `PostLayout` component:

```astro
---
import PostLayout from '../../layouts/PostLayout.astro';
---

<PostLayout
  title="Your Post Title"
  description="Brief description"
  date="2025-01-01"
  tags={['Tag1', 'Tag2']}
>
  Your content here...
</PostLayout>
```

Then add it to the posts list in `src/pages/index.astro` and `src/pages/posts/index.astro`.

## License

MIT