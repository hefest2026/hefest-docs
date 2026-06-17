# Hefest Docs

Architecture and development documentation for the Hefest school event management platform (AIBEST 2026 Burgas). This is a MkDocs-Material site — source is in `docs/`, generated site in `site/`.

## Commands

```bash
uv run mkdocs serve    # Dev server at http://127.0.0.1:8000
uv run mkdocs build    # Build static site to site/
```

## Structure

- `docs/index.md` — Homepage
- `docs/superpowers/specs/` — Architecture and design specs
- `mkdocs.yml` — Site config, nav, theme, plugins

## Conventions

- Nav is managed manually in `mkdocs.yml` under the `nav:` key
- Mermaid diagrams are supported in any markdown file
