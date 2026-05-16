# kiok-docs

Documentation site for [kiok](https://github.com/cloudcheflabs) — a workflow
orchestrator for authoring, scheduling, and running DAGs.

Built with [MkDocs](https://www.mkdocs.org/) and the
[Material](https://squidfunk.github.io/mkdocs-material/) theme.

## Local preview

```bash
pip install mkdocs-material
mkdocs serve
```

Then open <http://localhost:8000>.

## Build

```bash
mkdocs build
```

The static site is written to `site/` (git-ignored).
