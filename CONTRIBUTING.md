# Contributing

Thanks for taking the time to contribute! This repository hosts central,
reusable **GitHub Actions workflows** (`on: workflow_call`), consumed by
other repositories of this account.

## Workflow

1. Create a feature branch from `main`.
2. Make your changes (see conventions below).
3. Run the linters locally (see [Linting](#linting)).
4. Open a pull request against `main`. The `Lint` workflow runs
   automatically and code owners (see `.github/CODEOWNERS`) are requested
   for review.

## Repository layout

Reusable workflows must live **flat** in `.github/workflows/` (GitHub does
not support subdirectories for them), so files are grouped by filename
prefix:

```text
.github/workflows/jekyll-*.yml   # Jekyll build/deploy/validate
.github/workflows/latex-*.yml    # LaTeX document builds
.github/workflows/python-*.yml   # Python static analysis
.github/workflows/rdf-*.yml      # RDF validation, generation and release
.github/workflows/repo-*.yml     # repository hygiene (lint, ...)
.github/workflows/misc-*.yml     # everything else
.github/workflows/lint.yml       # repository CI (not reusable)
.github/workflows/codeql.yml     # repository CI (not reusable)
```

Every reusable workflow must be documented in the [README](README.md)
with a usage snippet plus an inputs (and, where applicable, secrets)
table. Shared build steps belong in composite actions in
[`stklug84/actions`](https://github.com/stklug84/actions), never inlined
here twice.

## Conventions

### Reusable workflows

- Start the file with the YAML document start marker (`---`) followed by
  a comment header (`# Reusable workflow: ...`) describing the workflow
  and showing a consumption example
  (`uses: stklug84/github-workflows/.github/workflows/<file>@vX.Y.Z`).
- Use a bare `on:` key for triggers — `.yamllint.yml` configures the
  `truthy` rule globally, so no per-file disable comment is needed.
- Use 2-space indentation; keep lines at 120 characters or less
  (`.yamllint.yml`).
- Declare least-privilege `permissions:` in the workflow itself (called
  workflows do not inherit the caller's permissions). Document any scopes
  the caller must grant in the header comment and the README.
- Do **not** set a `concurrency:` group — that is the caller's decision.
- Expose tunables as typed `workflow_call` inputs with sensible defaults:
  a `runs-on` input (default `ubuntu-latest`) and `run-*` boolean inputs
  to toggle optional jobs.
- Never interpolate `${{ ... }}` expressions directly inside `run:`
  scripts. Pass them via `env:` and reference the environment variables
  instead (this keeps the scripts injection-safe and shellcheck-clean).
- Start every bash script with `set -eu` or `set -euo pipefail`.
- Pin third-party actions (everything outside `actions/` and `github/`)
  to a full commit SHA with a trailing version comment, e.g.
  `uses: owner/action@<sha>  # v2`. Dependabot bumps the SHA and keeps
  the comment in sync.

## Linting

Pull requests must pass the `Lint` workflow
(`.github/workflows/lint.yml`), which is a thin caller of the reusable
`repo-lint.yml` in this same repository — the catalog lints itself with
the workflow it publishes, so a regression there fails this repository's
own CI first. Run the same checks locally before pushing (requires
`actionlint`, `yamllint`, `npx`):

```bash
actionlint
yamllint --strict .
npx markdownlint-cli2 --config .markdownlint.yml '**/*.md'
```

| Check        | Configuration        | Scope                                   |
|--------------|----------------------|------------------------------------------|
| actionlint   | built-in             | `.github/workflows/*`                    |
| shellcheck   | `.shellcheckrc`      | inline `run:` bash (via actionlint)      |
| yamllint     | `.yamllint.yml`      | all YAML files                           |
| markdownlint | `.markdownlint.yml`  | all Markdown files                       |

There is no standalone shellcheck step: all bash in this repository lives
inside workflow `run:` blocks, which actionlint feeds through shellcheck
automatically.

## Code scanning

The `CodeQL` workflow (`.github/workflows/codeql.yml`) scans the
workflows with the `actions` language and the `security-extended` query
suite (expression injection, excessive permissions, unpinned action
tags, untrusted checkout, ...). It runs on pull requests against `main`,
on pushes to `main`, and weekly. Findings appear under the repository's
Security tab and must be fixed or explicitly dismissed with a
justification before merging.

## Versioning and releases

Releases are tagged `vX.Y.Z` with a moving major alias (`vX`):

- **Patch/minor** changes (fixes, new optional inputs): bump the version
  and move the major alias forward.
- **Breaking** changes (removed/renamed inputs or outputs, changed
  defaults with behavioral impact): bump the major version and create a
  new alias.

Consumers pin to the major alias (`@v1`), to an exact tag, or — for
repositories with CodeQL's unpinned-tag query enabled — to the commit
SHA of a release tag with a trailing version comment
(`@<sha>  # v1.3.0`).
