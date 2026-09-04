# TM1 REST API Documentation

Documentation site for the IBM Planning Analytics TM1 REST API, built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

## Getting started

Install dependencies:

```bash
pip install -r requirements.txt
```

Serve the site locally with live reload:

```bash
mkdocs serve
```

Build the static site:

```bash
mkdocs build
```

The generated output is written to `site/` and is not committed to the repository.

## Project structure

```
docs/               Markdown source pages
mkdocs.yml           Site configuration and navigation
requirements.txt     Python dependencies
```

To add a page, create a Markdown file under `docs/` and add it to the `nav` section of [mkdocs.yml](mkdocs.yml).

## Deployment

Pushes to `main` automatically build and publish the site to GitHub Pages via [.github/workflows/deploy.yml](.github/workflows/deploy.yml), which runs `mkdocs gh-deploy --force` to push the built site to the `gh-pages` branch.

To deploy manually instead:

```bash
mkdocs gh-deploy
```

## License

[MIT](LICENSE)
