# github-workflows

Central, reusable GitHub Actions workflows for this account.

## `deploy-pages.yml` — build & deploy a Jekyll site to GitHub Pages

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
    uses: stklug84/github-workflows/.github/workflows/deploy-pages.yml@v1
```

The caller **must** declare the `permissions` block above — reusable workflows
cannot elevate beyond what the caller grants.

### Requirements in the consuming repo

The calling repo must contain a `Gemfile` that includes Jekyll, for example:

```ruby
source 'https://rubygems.org'

gem 'jekyll'
gem 'github-pages'
gem 'webrick' # required for Ruby 3.0+
```

Commit a `Gemfile.lock` (and optionally a `.ruby-version`) for reproducible,
cache-friendly builds.

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
