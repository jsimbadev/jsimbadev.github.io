# Jordan Simbananiye

Personal website and blog for Jordan Simbananiye, published at
<https://jsimbadev.github.io/>.

The site collects short biographical pages, contact links, posts, and
presentation material related to geometry-aware Monte Carlo methods,
statistical computation, and research tooling.

## Toolkit

- [Hugo](https://gohugo.io/) for static site generation.
- [PaperMod](https://github.com/adityatelange/hugo-PaperMod) as a Hugo module
  theme.
- Markdown content under `content/`.
- Static assets under `static/`.
- Standalone Reveal.js slide decks under `static/slides/`, with Hugo pages under
  `content/presentations/` linking to and embedding them.
- GitHub Actions and GitHub Pages for production deployment.

Generated output in `public/` is intentionally ignored. The production build is
created in CI and uploaded as a GitHub Pages artifact.

## Local Development

Install Hugo Extended, then run:

```bash
hugo server
```

The local preview is usually available at <http://localhost:1313/>.

For a production-style local build:

```bash
hugo --gc --minify
```

This writes generated files to `public/`.

## Repository Layout

```text
content/                 Markdown pages, posts, and presentation entries
layouts/                 Local Hugo layout overrides
static/                  Images and standalone static files
static/slides/           Standalone Reveal.js presentation decks
hugo.toml                Site configuration
.github/workflows/       GitHub Pages build and deploy workflow
```

## Publishing

Work on feature branches and open pull requests into `main`. When changes are
merged into `main`, `.github/workflows/hugo.yaml` builds the Hugo site and
deploys it to GitHub Pages.

Typical edit flow:

```bash
git switch -c feature/my-change
git add <changed-files>
git commit -m "docs: describe the change"
git push -u origin feature/my-change
```

After the pull request is merged, GitHub Actions deploys the updated site.
