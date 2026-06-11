# github-workflows

[![Lint](https://github.com/stklug84/github-workflows/actions/workflows/lint.yml/badge.svg?event=pull_request)](https://github.com/stklug84/github-workflows/actions/workflows/lint.yml)
[![CodeQL](https://github.com/stklug84/github-workflows/actions/workflows/codeql.yml/badge.svg)](https://github.com/stklug84/github-workflows/actions/workflows/codeql.yml)

Central, reusable GitHub Actions workflows for this account.

> **Note:** Reusable workflows must live flat in `.github/workflows/` (GitHub
> does not support subdirectories for them), so files are grouped by filename
> prefix (`jekyll-*`, `misc-*`). The shared Jekyll build steps are provided as
> composite actions from [`stklug84/actions`](https://github.com/stklug84/actions).

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
    uses: stklug84/github-workflows/.github/workflows/jekyll-deploy-pages.yml@v1.2.0
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

No secrets are required — GitHub Pages deployment uses the OIDC `id-token`.

## `jekyll-validate-pages.yml` — build & advisory checks for pull requests

Builds the site with Bundler (the `build` job — the meaningful gate) and runs a
set of advisory checks (`html-validate`, `link-check`, `spell-check`,
`yaml-lint`, `actions-lint`, `markdown-lint`), each `continue-on-error`. Every
advisory job can be toggled off per repo.

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
    uses: stklug84/github-workflows/.github/workflows/jekyll-validate-pages.yml@v1.2.0
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
| `run-actions-lint`      | `true`               | Toggle the actions-lint job.                   |
| `run-markdown-lint`     | `true`               | Toggle the markdown-lint job.                  |

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
    uses: stklug84/github-workflows/.github/workflows/jekyll-deploy-preview.yml@v1.2.0
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
    uses: stklug84/github-workflows/.github/workflows/misc-pr-preview-comment.yml@v1.2.0
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
    uses: stklug84/github-workflows/.github/workflows/latex-build-cv.yml@v1.3.0
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

The TeX Live image is resolved from the caller's Dockerfile, so the caller's
Dependabot (docker ecosystem) keeps the digest pin current.

## `latex-build-book.yml` — LaTeX book PDF + EPUB

Builds the print-ready PDF (engine/aux-tool detection via
`stklug84/actions/texlive/detect`) and, unless toggled off, the validated
EPUB 3 edition, then combines both into a single user-facing artifact.

### Usage

```yaml
jobs:
  build:
    uses: stklug84/github-workflows/.github/workflows/latex-build-book.yml@v1.3.0
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

### Private repo access (one-time)

Because this repo is **private**, other private repos cannot call its workflows
until access is shared:

> **Settings → Actions → General → Access →**
> *"Accessible from repositories owned by the user/organization"*

### Versioning

Callers should pin to a tag:

- `@v1` — moving major tag, receives backwards-compatible updates.
- `@v1.0.0` — exact, fully pinned.

Maintainers: on each release, create `vX.Y.Z` and re-point the `vX` major tag.
