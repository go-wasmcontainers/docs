# docs

[![docs](https://github.com/go-wasmcontainers/docs/actions/workflows/docs.yml/badge.svg)](https://github.com/go-wasmcontainers/docs/actions/workflows/docs.yml)
[![License](https://img.shields.io/badge/license-BSD--3--Clause-0079A8?style=flat-square)](LICENSE)

**The go-wasmcontainers documentation site**, published at
<https://go-wasmcontainers.github.io/docs/>.

A [Hugo](https://gohugo.io/) site with the [tannevaled/hextra](https://github.com/tannevaled/hextra)
fork of the [Hextra](https://github.com/imfing/hextra) theme, pinned at its
**v0.20.0** tag, built the same way as the
[go-slides documentation](https://github.com/go-slides/docs), with
go-wasmcontainers' branding. It documents
[`go-wasmcontainers/engine`](https://github.com/go-wasmcontainers/engine),
which has no release yet: the pages describe the `main` branch, and mark
everything not written as **planned**.

## Versions

A version of the documentation is **the minor version of
[`go-wasmcontainers/engine`](https://github.com/go-wasmcontainers/engine) it
describes**, or `main` while no release is tagged:

| Source | Published under |
| --- | --- |
| branch `main`, with `params.engine.version` in `hugo.yaml` | `https://go-wasmcontainers.github.io/docs/<MAJOR.MINOR>/`, or `/docs/main/` while it says `main` |
| the newest release (else `main`) | also `https://go-wasmcontainers.github.io/docs/latest/` |

`https://go-wasmcontainers.github.io/docs/` redirects to `latest/`. At the
first release: set `params.engine.version` to its tag and check every page
against that release.

## Layout

| Path | What |
| --- | --- |
| `content/` | the pages; `_index.md` is the home, `weight` orders the sidebar |
| `i18n/en.yaml` | the site's strings, beside the theme's |
| `hugo.yaml` | `baseURL`, the engine version described, the navbar, the brand mounts |
| `layouts/_partials/custom/version-select.html` | the version selector (reads `versions.json`) |
| `layouts/_partials/custom/content-begin.html` | the outdated-version banner |
| `layouts/_partials/favicons.html` | the brand's favicons |
| `assets/css/custom.css` | the brand cyan as Hextra's primary colour, with its contrast measured |
| `themes/hextra` | submodule: tannevaled/hextra at tag v0.20.0 |
| `branding` | submodule: [go-wasmcontainers/brand](https://github.com/go-wasmcontainers/brand) |
| `scripts/publish-version.sh` | puts one build into a `gh-pages` checkout and rewrites `versions.json`, `latest` and the root redirect |

## Working locally

```sh
git clone --recurse-submodules https://github.com/go-wasmcontainers/docs.git
cd docs
mkdir -p data && sh themes/hextra/scripts/page-history.sh > data/pagehistory.json
hugo server --baseURL http://localhost:1313/docs/main/
```

Hugo 0.146 or later; no Python, no Node.

## Publication

`.github/workflows/docs.yml`:

| Job | When | What |
| --- | --- | --- |
| `build` | pull requests, `main` | builds with `--panicOnWarning`, checks internal links and anchors (lychee, offline), uploads the site |
| `links` | pull requests, `main` | fetches every external URL the pages name |
| `deploy` | `main` | `scripts/publish-version.sh` into `gh-pages` (started empty on the first deploy), then a commit and a push; write access on this job only |

GitHub Pages serves the `gh-pages` branch (set it once `gh-pages` exists).

## Licence

BSD-3-Clause.
