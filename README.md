# github-workflows

[![Lint](https://github.com/stklug84/github-workflows/actions/workflows/lint.yml/badge.svg?event=pull_request)](https://github.com/stklug84/github-workflows/actions/workflows/lint.yml)
[![CodeQL](https://github.com/stklug84/github-workflows/actions/workflows/codeql.yml/badge.svg)](https://github.com/stklug84/github-workflows/actions/workflows/codeql.yml)

Central, reusable GitHub Actions workflows for this account.

> **Note:** Reusable workflows must live flat in `.github/workflows/` (GitHub
> does not support subdirectories for them), so files are grouped by filename
> prefix (`jekyll-*`, `latex-*`, `python-*`, `rdf-*`, `release-*`, `repo-*`,
> `misc-*`). The shared Jekyll, TeX Live, Python and RDF build/check steps are
> provided as composite actions from
> [`stklug84/actions`](https://github.com/stklug84/actions).

## `jekyll-deploy-pages.yml` — build & deploy a Jekyll site to GitHub Pages

A reusable workflow (`on: workflow_call`) that builds a Jekyll site with
**Bundler** (`bundle exec jekyll build`, driven by the calling repo's own
`Gemfile`) and deploys it to GitHub Pages via the official Pages actions.

### Usage

In a consuming repository, replace the build/deploy logic with a thin caller:

```yaml
name: Deploy Jekyll with GitHub Pages dependencies preinstalled

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  deploy:
    uses: stklug84/github-workflows/.github/workflows/jekyll-deploy-pages.yml@v1.5.0
```

The caller **must** declare the `permissions` block above — reusable workflows
cannot elevate beyond what the caller grants.

### Requirements in the consuming repo

The calling repo must contain a `Gemfile` that includes Jekyll, for example:

```ruby
source 'https://rubygems.org'

gem 'jekyll', '~> 4.3'
gem 'webrick' # required for Ruby 3.0+
```

Commit a `Gemfile.lock` and a `.ruby-version` (e.g. `3.3`) for reproducible,
cache-friendly builds. `ruby/setup-ruby` requires an explicit version, so a
`.ruby-version` (or `.tool-versions`) file is effectively required unless the
`ruby-version` input is set.

### Inputs

| Input              | Default          | Description                                                              |
|--------------------|------------------|--------------------------------------------------------------------------|
| `source`           | `./`             | Jekyll source directory.                                                 |
| `destination`      | `./_site`        | Build output directory (uploaded as the Pages artifact).                 |
| `runs-on`          | `ubuntu-latest`  | Runner label for both jobs.                                              |
| `environment-name` | `github-pages`   | Deployment environment name.                                            |
| `ruby-version`     | `""`             | Ruby version. Empty → resolved from `.ruby-version` / `Gemfile`.         |
| `checkout-ref`     | `""`             | Optional git ref to build. Defaults to the caller's ref.                 |
| `prebuild-artifact`| `""`             | Artifact (from a caller `generate` job) downloaded before the build. Empty → no-op. See [Pattern A](#pattern-a-prebuild-artifact-handoff). |
| `prebuild-into`    | `.`              | Directory the prebuild artifact is extracted into (e.g. `_data`).        |

No secrets are required — GitHub Pages deployment uses the OIDC `id-token`.

## `jekyll-validate-pages.yml` — build & quality checks for pull requests

Builds the site with Bundler (the `build` job — the primary gate) and runs a
set of quality checks (`html-validate`, `link-check`, `spell-check`,
`yaml-lint`, `actions-lint`, `markdown-lint`). Each job reports its real
pass/fail status (no `continue-on-error`), so a failing tool fails its check.
Consumers choose which checks are blocking by listing them as required status
checks in branch protection / a ruleset; any job can be toggled off per repo
via the `run-*` inputs.

### Usage

```yaml
name: PR validation

on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

permissions:
  contents: read

concurrency:
  group: pr-validate-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  validate:
    uses: stklug84/github-workflows/.github/workflows/jekyll-validate-pages.yml@v1.5.0
```

> **Branch protection:** the gating check is exposed as `<caller-job> / build`
> (e.g. `validate / build`). Point your required status check at that name.

### Inputs

| Input                   | Default              | Description                                   |
|-------------------------|----------------------|-----------------------------------------------|
| `runs-on`               | `ubuntu-latest`      | Runner label for all jobs.                     |
| `source` / `destination`| `./` / `./_site`     | Jekyll build paths.                            |
| `ruby-version`          | `""`                 | Ruby version (empty → `.ruby-version`/Gemfile).|
| `node-version`          | `20`                 | Node for html-validate.                        |
| `html-validate-version` | `9`                  | html-validate npm version.                     |
| `cspell-config`         | `.cspell.json`       | cspell config path.                            |
| `yamllint-config`       | `.yamllint.yml`      | yamllint config path.                          |
| `yamllint-paths`        | `.github/ _config.yml` | Space-separated yamllint targets.            |
| `markdownlint-globs`    | `**/*.md`            | markdownlint-cli2 globs.                       |
| `run-html-validate`     | `true`               | Toggle the html-validate job.                  |
| `run-link-check`        | `true`               | Toggle the link-check job.                     |
| `run-spell-check`       | `true`               | Toggle the spell-check job.                    |
| `run-yaml-lint`         | `true`               | Toggle the yaml-lint job.                      |
| `run-actions-lint`      | `true`               | Toggle the actions-lint job.                  |
| `run-markdown-lint`     | `true`               | Toggle the markdown-lint job.                  |
| `prebuild-artifact`     | `""`                 | Artifact (from a caller `generate` job) downloaded before the build. Empty → no-op. See [Pattern A](#pattern-a-prebuild-artifact-handoff). |
| `prebuild-into`         | `.`                  | Directory the prebuild artifact is extracted into (e.g. `_data`). |

## `jekyll-deploy-preview.yml` — per-commit preview publishing

Builds a preview (with `baseurl: /<short-sha>`) and publishes it into a separate
previews repository under a short-SHA path, pruning to the newest N previews.

### Usage

```yaml
name: Deploy per-commit preview

on:
  push:
    branches: ['**']
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: preview-${{ github.ref }}
  cancel-in-progress: false

jobs:
  preview:
    uses: stklug84/github-workflows/.github/workflows/jekyll-deploy-preview.yml@v1.5.0
    with:
      previews-repo: owner/my-previews
      preview-domain: https://example.com
    secrets:
      previews-deploy-token: ${{ secrets.PREVIEWS_DEPLOY_TOKEN }}
```

### Inputs & secrets

| Input                   | Default     | Description                                     |
|-------------------------|-------------|-------------------------------------------------|
| `previews-repo`         | *(required)*| `owner/repo` to publish previews into.          |
| `preview-domain`        | *(required)*| Base domain (no trailing slash).                |
| `keep-last`             | `20`        | Newest preview directories to retain.           |
| `environment-name`      | `previews`  | Deployment environment.                         |
| `source` / `destination`| `./` / `./_site` | Jekyll build paths.                        |
| `ruby-version`          | `""`        | Ruby version (empty → `.ruby-version`/Gemfile). |

| Secret                  | Required | Description                                       |
|-------------------------|----------|---------------------------------------------------|
| `previews-deploy-token` | yes      | Token with write access to the previews repo.     |

## `misc-pr-preview-comment.yml` — sticky PR preview link

Upserts a sticky pull-request comment linking to the per-commit preview URL.

### Usage

```yaml
name: PR preview URL comment

on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write

concurrency:
  group: pr-preview-comment-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  comment:
    uses: stklug84/github-workflows/.github/workflows/misc-pr-preview-comment.yml@v1.5.0
    with:
      preview-domain: https://example.com
```

The caller **must** grant `pull-requests: write`.

### Inputs

| Input            | Default       | Description                                |
|------------------|---------------|--------------------------------------------|
| `preview-domain` | *(required)*  | Base domain (no trailing slash).           |
| `comment-header` | `preview-url` | Sticky comment header key (for upserting).  |

## `latex-build-cv.yml` — matrix-build LaTeX CV variants

Discovers every CV variant under `<root>/*/` (via
`stklug84/actions/texlive/discover-variants`), matrix-builds each one inside
the caller's digest-pinned TeX Live container, and combines all PDFs into a
single artifact named after the calling repo. No filenames are hardcoded.
Optionally (`release: "true"`) publishes the PDFs as a versioned GitHub
release (tag `v<YYYY.MM.DD>-r<run-number>`) on push events, pruned to the
`release-keep` newest releases.

### Usage

```yaml
name: Build Document

on:
  pull_request:
  workflow_dispatch:
    inputs:
      local:
        description: "Set to 'true' when running locally via gh act"
        required: false
        default: "false"
        type: string

permissions:
  contents: read

concurrency:
  group: build-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    uses: stklug84/github-workflows/.github/workflows/latex-build-cv.yml@v1.5.0
    with:
      texinputs: ".:../..:../../styles:../../images:"
      local: ${{ inputs.local }}
```

### Inputs

| Input                | Default                              | Description                                          |
|----------------------|--------------------------------------|------------------------------------------------------|
| `texinputs`          | `.:../..:../../styles:../../images:` | TEXINPUTS path, relative to each variant directory.  |
| `root`               | `cvs`                                | Directory scanned for CV variants.                   |
| `default-engine`     | `latexmk`                            | Engine for variants without an `.engine` file.       |
| `local`              | `false`                              | gh-act local mode (skips artifact upload).           |
| `texlive-dockerfile` | `.github/docker/texlive/Dockerfile`  | Caller file whose `FROM` pins the TeX Live image.    |
| `runs-on`            | `ubuntu-latest`                      | Runner label for all jobs.                           |
| `release`            | `false`                              | Publish PDFs as a versioned GitHub release (push only). |
| `release-keep`       | `0`                                  | Keep only the N newest releases (`0` = keep all).    |
| `prebuild-artifact`  | `""`                                 | Artifact (from a caller `generate` job) downloaded after checkout, before build. Empty → no-op. See [Pattern A](#pattern-a-prebuild-artifact-handoff). |
| `prebuild-into`      | `.`                                  | Directory the prebuild artifact is extracted into (default repo root). |

The TeX Live image is resolved from the caller's Dockerfile, so the caller's
Dependabot (docker ecosystem) keeps the digest pin current. With
`release: "true"` the caller must grant `contents: write` on the calling
job; otherwise `contents: read` suffices.

## `latex-build-book.yml` — LaTeX book PDF + EPUB

Builds the print-ready PDF (engine/aux-tool detection via
`stklug84/actions/texlive/detect`) and, unless toggled off, the validated
EPUB 3 edition, then combines both into a single user-facing artifact.

### Usage

```yaml
jobs:
  build:
    uses: stklug84/github-workflows/.github/workflows/latex-build-book.yml@v1.5.0
    with:
      book-title: "My Book"
      rasterize-pdf: images/map.pdf
      stylesheet: config/ebook.css
      epubcheck-filter-file: config/epubcheck-filter.txt
```

### Inputs

| Input                   | Default                | Description                                           |
|-------------------------|------------------------|-------------------------------------------------------|
| `book-title`            | *(required)*           | Title patched into the EPUB metadata.                 |
| `engine`                | `""`                   | latexmk / pdflatex / latex-chain (empty → default).   |
| `local`                 | `false`                | gh-act local mode (skips artifact upload).            |
| `main-tex`              | `""`                   | Main basename (empty → `\documentclass` auto-detect). |
| `eps-from-pdf`          | `""`                   | PDF graphics converted to `.eps` for latex-chain (DVI). |
| `texlive-image`         | `texlive/texlive:latest` | Container image for the build jobs.                 |
| `run-epub`              | `true`                 | Toggle the EPUB build + validation jobs.              |
| `epub-config`           | `config/ebook.cfg`     | tex4ebook configuration file.                         |
| `epub-build-file`       | `config/ebook.mk4`     | tex4ebook build file.                                 |
| `stylesheet`            | `""`                   | CSS bundled into the EPUB.                            |
| `images-dir`            | `images`               | Images bundled into the EPUB.                         |
| `rasterize-pdf`         | `""`                   | Optional vector PDF rasterized to 300 dpi PNG.        |
| `fonts`                 | `""`                   | Newline-separated OTFs embedded via kpsewhich.        |
| `font-license`          | `""`                   | License file bundled with the fonts.                  |
| `epubcheck-version`     | `5.1.0`                | Pinned epubcheck release.                             |
| `epubcheck-filter-file` | `""`                   | Optional epubcheck findings filter.                   |
| `artifact-name`         | `""`                   | Combined artifact name (empty → repo name).           |
| `runs-on`               | `ubuntu-latest`        | Runner label for all jobs.                            |

## `python-validate.yml` — strict Python static analysis

Runs Ruff (lint + format check), mypy (strict type check) and Bandit
(security scan) over the given paths, via the composite actions in
[`stklug84/actions/python`](https://github.com/stklug84/actions). Each job
reports its real pass/fail status (no `continue-on-error`); consumers choose
which checks are blocking by listing them as required status checks in branch
protection / a ruleset, and any job can be toggled off via the `run-*` inputs.

### Usage

```yaml
name: PR validation

on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

permissions:
  contents: read

concurrency:
  group: python-validate-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  validate:
    uses: stklug84/github-workflows/.github/workflows/python-validate.yml@v1.18.0
    with:
      paths: "scripts tests"
      mypy-extra-deps: "types-PyYAML Jinja2"
```

### Inputs

| Input               | Default         | Description                                                    |
|---------------------|-----------------|----------------------------------------------------------------|
| `runs-on`           | `ubuntu-latest` | Runner label for all jobs.                                     |
| `paths`             | `.`             | Space-separated files/directories to analyze.                  |
| `config`            | `""`            | Shared tool config (e.g. `pyproject.toml`), forwarded to all three tools. Empty → per-tool auto-discovery. |
| `ruff-version`      | `0.16.0`        | Ruff release (mirrors the action's pin).                       |
| `check-format`      | `true`          | Also run `ruff format --check` (string `'true'`/`'false'`).    |
| `mypy-version`      | `1.19.0`        | mypy release (mirrors the action's pin).                       |
| `mypy-strict`       | `true`          | Pass `--strict` to mypy (string `'true'`/`'false'`).           |
| `mypy-extra-deps`   | `""`            | Space-separated pip packages installed alongside mypy (stubs / imports). |
| `bandit-version`    | `1.8.0`         | Bandit release, with the toml extra (mirrors the action's pin). |
| `bandit-severity`   | `low`           | Minimum severity reported (`low`/`medium`/`high`).             |
| `bandit-confidence` | `low`           | Minimum confidence reported (`low`/`medium`/`high`).           |
| `run-ruff`          | `true`          | Toggle the ruff job.                                           |
| `run-mypy`          | `true`          | Toggle the mypy job.                                           |
| `run-bandit`        | `true`          | Toggle the bandit job.                                         |

## `rdf-validate.yml` — Turtle, SPARQL and OWL validation

Validates Turtle syntax (Apache Jena `riot --validate`), SPARQL query syntax
(rdflib `prepareQuery` — SPARQL 1.1 *Query* grammar only, Update requests are
not covered) and OWL consistency/coherence (`robot reason`), via the composite
actions in [`stklug84/actions/rdf`](https://github.com/stklug84/actions). Each
job reports its real pass/fail status (no `continue-on-error`) and can be
toggled off via the `run-*` inputs. The `owl` job additionally requires a
non-empty `owl-files` list and is skipped otherwise, so Turtle/SPARQL-only
repositories work with the defaults.

### Usage

```yaml
name: PR validation

on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

permissions:
  contents: read

concurrency:
  group: rdf-validate-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  validate:
    uses: stklug84/github-workflows/.github/workflows/rdf-validate.yml@v1.21.0
    with:
      owl-files: "ontology/core.ttl"
```

### Inputs

| Input            | Default         | Description                                                       |
|------------------|-----------------|-------------------------------------------------------------------|
| `runs-on`        | `ubuntu-latest` | Runner label for all jobs.                                        |
| `turtle-glob`    | `**/*.ttl`      | Glob pattern(s) selecting Turtle files (find-style).              |
| `jena-version`   | `5.4.0`         | Apache Jena release (mirrors the action's pin).                   |
| `java-version`   | `21`            | JDK version for the turtle and owl jobs.                          |
| `sparql-glob`    | `**/*.rq`       | Glob pattern(s) selecting SPARQL files (pathlib-style).           |
| `python-version` | `3.12`          | Python version for the sparql job.                                |
| `rdflib-version` | `>=7,<8`        | PEP 440 specifier for rdflib (mirrors the action's default).      |
| `owl-files`      | `""`            | Space-separated ontology files to reason over. Empty → owl job skipped. |
| `owl-reasoner`   | `hermit`        | Reasoner for `robot reason` (hermit/elk/whelk/jfact/structural).  |
| `owl-catalog`    | `""`            | Optional OASIS XML catalog for `robot --catalog` (owl:imports via urn: etc.). |
| `owl-prepare-command` | `""`       | Optional shell command run before reasoning (e.g. regenerate an untracked catalog). Empty skips the step. |
| `robot-version`  | `1.9.8`         | ROBOT release, pinned robot.jar (mirrors the action's pin).       |
| `run-turtle`     | `true`          | Toggle the turtle job.                                            |
| `run-sparql`     | `true`          | Toggle the sparql job.                                            |
| `run-owl`        | `true`          | Toggle the owl job (also requires non-empty `owl-files`).         |

## `rdf-generate.yml` — regenerate RDF instance graphs from an inventory

Regenerates RDF instance graphs from a source inventory via the
[`stklug84/actions/rdf/generate-individuals`](https://github.com/stklug84/actions)
composite action: pinned Python + rdflib, a cached remote API response
directory (`actions/cache`), and the generator's `fetch` → `generate` →
`collection` pipeline with optional repo-specific prepare/finalize commands.
When the regenerated files differ from the checked-out state, the workflow
commits them to a branch and opens (or force-updates) a pull request —
or uploads them as a workflow artifact with `create-pr: false`.

> **Precondition:** the generator's **source inventory must be committed**.
> Every stage reads it, so a repository that keeps its inventory untracked
> (a private export, a gitignored dump) cannot use this workflow — the
> generator exits before the first stage and there is no runner-side
> substitute. Check the inventory in, or regenerate locally.

Pull requests are created with the workflow token by default; pass the
`token` secret (PAT or GitHub App token) when downstream workflows must
trigger on the created pull request — events raised with the built-in
`GITHUB_TOKEN` do not start new workflow runs.

### Usage

```yaml
name: Regenerate knowledge graph

on:
  push:
    branches: [main]
    paths: [collection.csv]
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: rdf-generate
  cancel-in-progress: false

jobs:
  generate:
    uses: stklug84/github-workflows/.github/workflows/rdf-generate.yml@v2.0.0
    permissions:
      contents: write
      pull-requests: write
    with:
      generator: "scripts/generate_individuals.py"
      cache-hash-files: "inventory.csv"
      commit-paths: "sets aggregate.ttl"
      prepare-command: "python3 scripts/build_vocab.py"
      finalize-command: "python3 scripts/update_imports.py"
```

### Inputs & secrets

| Input              | Default                                              | Description                                                    |
|--------------------|------------------------------------------------------|----------------------------------------------------------------|
| `runs-on`          | `ubuntu-latest`                                      | Runner label for all jobs.                                     |
| `generator`        | *(required)*                                         | Generator script, run as `python3 <generator> <stage>`.        |
| `prepare-command`  | `""`                                                 | Shell command before `fetch`; empty skips.                     |
| `finalize-command` | `""`                                                 | Shell command after `collection`; empty skips.                 |
| `python-version`   | `3.12`                                               | Python version for `actions/setup-python`.                     |
| `rdflib-version`   | `>=7,<8`                                             | PEP 440 specifier for rdflib; empty skips the install.         |
| `cache-dir`        | `/tmp/rdf-generate-cache`                            | API response cache directory (saved/restored).                 |
| `cache-key`        | `rdf-generate-cache`                                 | Cache key prefix.                                              |
| `cache-hash-files` | *(required)*                                         | `hashFiles` pattern(s) mixed into the cache key — the committed inventory. |
| `run-fetch`        | `true`                                               | Toggle the fetch stage (string, forwarded to the action).      |
| `run-generate`     | `true`                                               | Toggle the generate stage (string, forwarded to the action).   |
| `run-collection`   | `true`                                               | Toggle the collection stage (string, forwarded to the action). |
| `commit-paths`     | *(required)*                                         | Paths committed to the PR / packed into the artifact.          |
| `pr-branch`        | `chore/regenerate-graph`                             | Head branch for the regeneration pull request.                 |
| `pr-title`         | `chore(graph): regenerate instance graphs from inventory` | Pull request title.                                       |
| `commit-message`   | `chore(graph): regenerate instance graphs from inventory` | Commit message.                                           |
| `create-pr`        | `true`                                               | Open/update a PR; `false` uploads an artifact instead.         |
| `artifact-name`    | `regenerated-graph`                                  | Artifact name when `create-pr: false`.                         |

| Secret  | Required | Description                                                                  |
|---------|----------|------------------------------------------------------------------------------|
| `token` | no       | Push/PR token; falls back to the workflow token (which does not trigger CI). |

## `release-artifact.yml` — publish a validated artifact as a GitHub Release

Publishes a build artifact — a graph bundle, a Python wheel, anything a tag
should ship — as a versioned GitHub Release on a tag push, **after** the
artifact has been validated. The workflow owns the sequence and the guardrails
— tag-format check, Python setup, install, build, unpack, staged verification,
release — while every repository-specific step arrives as a shell command
input:

```text
check tag → setup python → install → prepare-command → build-command
  → unpack → stage-command → smoke-command → validate-command
  → gh release create
```

Each hook runs as its own named step, so a failure points at the stage that
produced it. Every command runs through `bash -c` with `RELEASE_TAG` (the full
tag), `RELEASE_VERSION` (the tag with `tag-prefix` stripped) and `BUNDLE_DIR`
(the unpack directory) exported.

The caller owns the trigger — a reusable workflow cannot declare one — and must
grant `contents: write`, since called workflows do not inherit the caller's
permissions.

### Usage

```yaml
name: Release graph bundle

on:
  push:
    tags: ["graph-*"]

permissions:
  contents: write

jobs:
  release:
    uses: stklug84/github-workflows/.github/workflows/release-artifact.yml@v3.0.0
    permissions:
      contents: write
    with:
      tag-prefix: "graph-"
      version-pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}$'
      install-command: 'pip install ".[validate]"'
      build-command: 'python3 scripts/build_graph_bundle.py --graph-version "$GRAPH_VERSION"'
      unpack-glob: "dist/*.tar.gz"
      validate-command: 'mtg-validate --check ttl --check consistency "$BUNDLE_DIR"'
      release-name: "Knowledge graph"
```

### Inputs

| Input                     | Default         | Description                                                       |
|---------------------------|-----------------|-------------------------------------------------------------------|
| `runs-on`                 | `ubuntu-latest` | Runner label for the job.                                         |
| `python-version`          | `3.12`          | Python version for `actions/setup-python`.                        |
| `tag-prefix`              | `""`            | Prefix stripped from the tag to obtain `RELEASE_VERSION`.         |
| `version-pattern`         | `""`            | `grep -E` pattern the stripped version must match. Empty → no check. |
| `install-command`         | `""`            | Installs the build/validation tooling. Empty skips the step.      |
| `prepare-command`         | `""`            | Runs after install, before build (uncommittable inputs). Empty skips. |
| `build-command`           | *(required)*    | Produces the release artifacts.                                   |
| `unpack-glob`             | `""`            | Glob for the gzip tar archive to unpack; must match exactly one file. Empty skips unpacking. |
| `unpack-dir`              | `/tmp/bundle`   | Directory the archive is unpacked into (`BUNDLE_DIR`).            |
| `unpack-strip-components` | `1`             | `tar --strip-components` value.                                   |
| `stage-command`           | `""`            | Runs after unpacking (stage extra inputs, echo the manifest). Empty skips. |
| `smoke-command`           | `""`            | Exercises the unpacked artifact the way a consumer would. Empty skips. |
| `validate-command`        | `""`            | Validates the unpacked artifact standalone. Empty skips.          |
| `release-assets`          | `dist/*`        | Space-separated globs attached to the release; must match ≥ 1 file. |
| `release-name`            | `""`            | Title prefix; rendered as `<release-name> <RELEASE_VERSION>`. Empty → gh uses the tag. |
| `generate-notes`          | `true`          | Pass `--generate-notes`.                                          |
| `draft`                   | `false`         | Create the release as a draft.                                    |
| `prerelease`              | `false`         | Mark the release as a prerelease.                                 |

## `repo-lint.yml` — repository hygiene linting

The lint trio every repository of this account runs — **actionlint** (workflow
syntax, including shellcheck over `run:` blocks), **yamllint** and
**markdownlint** — plus an optional **hadolint** job for repositories that ship
a Dockerfile. The linters read the calling repository's own configuration
(`.yamllint.yml`, `.markdownlint.yml`, `.shellcheckrc`); this workflow supplies
the runners and the pinned tool versions.

Each job reports its real pass/fail status (no `continue-on-error`) and can be
toggled off via the `run-*` inputs. The `hadolint` job additionally requires a
non-empty `hadolint-dockerfile` and is skipped otherwise, so repositories
without a Dockerfile work with the defaults. Repository-specific linting (a
shellcheck sweep over standalone scripts, schema validation, …) stays in the
caller as extra jobs alongside the `uses:` job.

### Usage

```yaml
name: Lint

on:
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: lint-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    uses: stklug84/github-workflows/.github/workflows/repo-lint.yml@v3.0.0
    with:
      hadolint-dockerfile: ".github/docker/texlive/Dockerfile"
      markdownlint-globs: |
        **/*.md
        !node_modules
```

### Inputs

| Input                 | Default             | Description                                                   |
|-----------------------|---------------------|---------------------------------------------------------------|
| `runs-on`             | `ubuntu-latest`     | Runner label for all jobs.                                    |
| `actionlint-version`  | `1.7.10`            | actionlint release to download.                               |
| `yamllint-version`    | `""`                | PEP 440 specifier appended to `pip install yamllint` (e.g. `==1.35.1`). Empty → latest. |
| `yamllint-paths`      | `.`                 | Space-separated paths passed to yamllint.                     |
| `yamllint-strict`     | `true`              | Run with `--strict` (warnings, line-length among them, fail). |
| `markdownlint-config` | `.markdownlint.yml` | markdownlint-cli2 configuration file.                         |
| `markdownlint-globs`  | `**/*.md`           | Newline-separated globs; exclusions are negation globs (`!.venv`). |
| `hadolint-dockerfile` | `""`                | Dockerfile to lint. Empty → hadolint job skipped.             |
| `run-actionlint`      | `true`              | Toggle the actionlint job.                                    |
| `run-yamllint`        | `true`              | Toggle the yamllint job.                                      |
| `run-markdownlint`    | `true`              | Toggle the markdownlint job.                                  |
| `run-hadolint`        | `true`              | Toggle the hadolint job (also requires `hadolint-dockerfile`). |

## `repo-codeql.yml` — CodeQL code scanning

CodeQL analysis matrixed over the languages the caller supplies. Every
repository of this account previously ran a hand-copied version of the same
three steps (checkout → init → analyze), and the copies had already drifted
apart on structure and weekly schedule.

The caller owns the trigger — a reusable workflow cannot declare one — and
must grant `security-events: write`, since called workflows do not inherit
the caller's permissions.

`languages` is a JSON array so it can drive the matrix; each entry reports its
own `Analyze (<language>)` check, which is the name a required status check
must reference.

`build-mode: none` suits interpreted and non-compiled targets (`actions`,
`python`, `javascript-typescript`, `ruby`). Compiled languages need
`autobuild` or a manual build, which this workflow deliberately does not
model.

### Usage

```yaml
name: CodeQL

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  schedule:
    - cron: '30 5 * * 1'

permissions:
  contents: read

jobs:
  codeql:
    uses: stklug84/github-workflows/.github/workflows/repo-codeql.yml@v3.0.0
    permissions:
      security-events: write
      contents: read
      actions: read
    with:
      languages: '["actions", "python"]'
```

### Inputs

| Input           | Default             | Description                                                    |
|-----------------|---------------------|-----------------------------------------------------------------|
| `runs-on`       | `ubuntu-latest`     | Runner label for the analysis job.                             |
| `languages`     | `'["actions"]'`     | JSON array of CodeQL languages; drives the job matrix.         |
| `queries`       | `security-extended` | Query suite for `codeql-action/init`.                          |
| `build-mode`    | `none`              | CodeQL build mode; compiled languages need `autobuild`.        |
| `config-file`   | `""`                | Optional CodeQL config file (query filters, path exclusions).  |
| `fail-on-error` | `false`             | Fail fast on the first failing matrix entry.                   |

## Pattern A: prebuild artifact handoff

`latex-build-cv.yml`, `jekyll-deploy-pages.yml`, and `jekyll-validate-pages.yml`
support an optional **prebuild artifact handoff**: a separate `generate` job in
the caller produces files (e.g. generated `.tex` variants or a `_data/cv.yml`),
uploads them as an artifact, and the reusable workflow downloads that artifact
**after its own checkout** and **before the build/deploy step**.

Both inputs are optional and default to a no-op (`prebuild-artifact: ""`), so
existing callers are unaffected:

| Input               | Default | Description                                                       |
|---------------------|---------|-------------------------------------------------------------------|
| `prebuild-artifact` | `""`    | Name of the artifact to download. Empty → step skipped (no-op).   |
| `prebuild-into`     | `.`     | Directory the artifact tree is extracted into (relative to repo). |

The artifact's internal file tree is laid down relative to `prebuild-into`, so
the artifact must contain files at their final paths.

### LaTeX-CV example

With `prebuild-into: "."` (repo root) the artifact contains generated files
already under their variant directories — e.g.
`cvs/photo-2page/cv-experience.tex`, `cvs/photo-2page/personal-info.tex`,
`cvs/sidebar/cv-experience.tex`, `cvs/sidebar/personal-info.tex`:

```yaml
name: Build Document

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: build-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Generate CV .tex files
        run: ./scripts/generate-cvs.sh   # writes cvs/<variant>/*.tex
      - name: Upload generated sources
        uses: actions/upload-artifact@v7
        with:
          name: cv-prebuild
          path: |
            cvs/**/cv-experience.tex
            cvs/**/personal-info.tex
          if-no-files-found: error

  build:
    needs: generate
    uses: stklug84/github-workflows/.github/workflows/latex-build-cv.yml@v1.6.0
    with:
      texinputs: ".:../..:../../styles:../../images:"
      prebuild-artifact: cv-prebuild
      prebuild-into: "."
```

### Jekyll example

Here the artifact contains a generated `cv.yml` and is extracted into `_data`,
so the build sees `_data/cv.yml`:

```yaml
name: Deploy Jekyll with generated data

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Generate cv.yml
        run: ./scripts/generate-cv-yml.sh   # writes cv.yml
      - name: Upload generated data
        uses: actions/upload-artifact@v7
        with:
          name: cv-data
          path: cv.yml
          if-no-files-found: error

  deploy:
    needs: generate
    uses: stklug84/github-workflows/.github/workflows/jekyll-deploy-pages.yml@v1.6.0
    with:
      prebuild-artifact: cv-data
      prebuild-into: "_data"
```

The same `prebuild-artifact` / `prebuild-into` inputs work identically for
`jekyll-validate-pages.yml` (PR validation builds against the generated
`_data/cv.yml`).

## Versioning

This repo is **public**, so its workflows are callable from any repository
without further access configuration. Callers should pin to a tag:

- `@v1` — moving major tag, receives backwards-compatible updates.
- `@v1.5.0` — exact, fully pinned.
- `@<commit-sha>  # v1.5.0` — SHA of the release tag with a version
  comment, for callers with CodeQL's unpinned-tag query enabled.

Maintainers: on each release, create `vX.Y.Z` and re-point the `vX` major tag.
