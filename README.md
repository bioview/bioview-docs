# bioview-docs

Documentation for [BioView](https://bioview.readthedocs.io), built with MkDocs
Material and published by Read the Docs.

```bash
poetry install
poetry run mkdocs serve
```

Pages live under `bioview-docs/`; the navigation tree is in `mkdocs.yml`.
`overrides/main.html` supplies the announcement bar, which reads the current
release from `extra.bioview_version` in `mkdocs.yml`.

## Versioning

Three places carry the release version, and all three are rewritten by
`release.sh` in the monorepo root along with every other version stamp:

- `pyproject.toml` — `version`
- `mkdocs.yml` — `extra.bioview_version`
- `bioview-docs/setup/downloads.md` and `bioview-docs/index.md` — the release
  badge and the installer file names

Do not bump them by hand; the point of routing them through `release.sh` is that
the site cannot fall behind the installers.

## Conventions

Design rationale lives here rather than in source comments — see
`bioview-docs/contributing/code-style.md`. Pages state what the code does and
why, with the numbers (queue depths, timeouts, defaults) matching the constants
they describe.
