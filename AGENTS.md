# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this repo is

A documentation-only site for the IBM Planning Analytics TM1 REST API, built with MkDocs Material. There is no application code — every change is either a Markdown content edit under `docs/` or a configuration edit to `mkdocs.yml`.

## Structure

- `docs/` — Markdown source pages, organized by section (`getting-started/`, `concepts/`, `tutorials/`, `api/`, `odata/`, `cookbook/`).
- `mkdocs.yml` — site config and the `nav` tree that drives the left sidebar.
- `requirements.txt` — Python dependencies (`mkdocs-material`).
- `site/` — generated build output; never edit or commit it.

## Adding or editing pages

1. Create or edit the Markdown file under `docs/`.
2. If it's a new page, add an entry to the `nav` section of `mkdocs.yml` in the appropriate section, using the same path relative to `docs/`.
3. Keep section groupings in `mkdocs.yml` and the `docs/` folder layout in sync — a nav entry should always point at a real file.

## Verifying changes

Before considering a change done, build the site in strict mode so broken nav references or config errors fail loudly:

```bash
mkdocs build --strict
```

Remove the generated `site/` directory afterward rather than committing it (`.gitignore` already excludes it).

## Conventions

- Use ATX-style headings (`#`, `##`, ...), starting each page with a single `#` title.
- Prefer fenced code blocks with a language hint (` ```json `, ` ```bash `, ` ```http `) for API requests/responses and shell commands.
- Keep API reference pages (`docs/api/*.md`) focused on endpoint-level detail; put narrative/how-to content in `docs/tutorials/` or `docs/cookbook/`.
