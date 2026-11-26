# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is Arman Akbarian's **personal academic site** (`https://rmanak.github.io`, remote `rmanak/rmanak.github.io`). It was created from the **al-folio v1.x** template. `AGENTS.md`, `.github/copilot-instructions.md`, `docs/` and the test suite are upstream al-folio files written for al-folio _maintainers_. Several of their rules do not apply to this site. Where they conflict with this file, **this file wins**:

| Upstream doc says                                                                                      | On this site                                                                                                                                                                                                                                                                                                                                                                                         |
| ------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Effective baseurl is `/al-folio`; build with `--baseurl /al-folio`; dev URL `localhost:4000/al-folio/` | `baseurl` is **blank** and `url: https://rmanak.github.io`. Build with a plain `bundle exec jekyll build`. **Never** pass `--baseurl /al-folio`. Internal links must be root-relative (`/blog/`). Use `{{ '/path' \| relative_url }}`, not `/al-folio/...` (some template content, e.g. `_pages/about_einstein.md`, still hardcodes `/al-folio/`).                                                   |
| "Stop sign": never create `_layouts/`, `_includes/`, `_sass/` …                                        | A personal site **may** shadow gem-owned files by recreating the same path locally (see `docs/ARCHITECTURE.md` → "Local overrides: your site vs. this repo"). Afterwards run `bundle exec al-folio upgrade overrides audit` and commit `.al-folio-overrides.yml`. Caveat: `npm run lint:style-contract` (run in `unit-tests.yml`) will then fail, because it enforces the upstream starter boundary. |
| Integration tests and visual regression are required gates                                             | These tests target upstream al-folio. Several hardcode `/al-folio` (`test/integration_{comments,distill,upgrade_cli}.sh`, `test/visual/*`), so failures may not mean this site is broken. Treat a successful `jekyll build` plus checking the rendered page as the real check.                                                                                                                       |

What still applies from upstream: the theme runtime lives in gems, not in this repo. Layouts, includes, Sass, Liquid tags and feature JS are in versioned `al_*` gems (`theme: al_folio_core`). The two silent failure modes described in `AGENTS.md` / `docs/ARCHITECTURE.md` also still apply:

- A feature renders only when (a) its gem is in the `Gemfile`, (b) it is listed under `plugins:` in `_config.yml`, (c) its flag is enabled, and (d) the page opts in through front matter. If any one is missing, the Liquid tag emits an empty string with no error.
- `Gemfile` (`group :al_folio_plugins`, exact pins) and `_config.yml` `plugins:` are two lists that must agree. Gem ids use underscores (`al_folio_core`); the repos use hyphens.

To find which gem owns a tag or layout, see `docs/BOUNDARIES.md` and `docs/ARCHITECTURE.md`.

## Content layout (site-specific)

- `_pages/`: top-level pages (`about.md` is the home page). Template pages the owner doesn't want are moved into `_pages/_template_pages/`. Jekyll skips paths that start with `_`, so they are not published. To restore a page, move it back to `_pages/`. Don't delete it.
- `_config.yml` `exclude:` also lists template-only pages (e.g. `_pages/about_einstein.md`).
- Collections: `_posts`, `_news`, `_projects`, `_teachings`, `_books`. Data lives in `_data/` (`cv.yml`, `socials.yml`, `coauthors.yml`, `venues.yml`, …) and `_bibliography/papers.bib`. Much of this is still al-folio demo content.
- Profile image: `assets/img/prof_pic.jpg`.
- `../old/` (outside this repo) holds the previous pre-al-folio site, both as a tarball and extracted. Use it only as source material when porting old content.

## Local dev

Ruby/Bundler are **not installed on the host**, so use Docker:

```bash
docker compose up -d          # builds/pulls amirpourmand/al-folio, runs bin/entry_point.sh
docker compose logs --tail=80
docker compose down
```

- **Port warning:** `docker-compose.yml` publishes `8080:8080`. That binds to all interfaces and **collides with the local llama-server on :8080**. When serving, map to localhost on another port (e.g. `127.0.0.1:4000:8080` and `127.0.0.1:35729:35729`), then browse `http://127.0.0.1:4000/`.
- The container writes output to container-local `/tmp/_site`, not to the bind-mounted `_site/`, to avoid bind-mount write deadlocks. It restarts Jekyll itself when `_config.yml` changes, because `--watch` does not hot-reload config.
- If you get `Permission denied … .jekyll-cache`, fill in the `build.args` UID/GID block in `docker-compose.yml`.

Lint (Node is available):

```bash
npm ci
npm run lint:prettier          # Prettier + @shopify/prettier-plugin-liquid, printWidth 150; `npx prettier . --write` fixes
npm run lint:style-contract    # upstream boundary check (see caveat above)
```

Optional Python tooling (CV rendering via `rendercv`, `bin/update_scholar_citations.py`, nbconvert for Jupyter posts) is listed in `requirements.txt`. Install it into a dedicated conda env, not system/base Python. `bin/setup-python-deps` uses `pip --user --break-system-packages`, so don't run it here.

## Deploy

Pushing to `main` triggers `.github/workflows/deploy.yml`, which runs `JEKYLL_ENV=production bundle exec jekyll build`, then `purgecss`, then force-pushes `_site` to the `gh-pages` branch. Deploy runs only when the changed paths match its filter (content, `assets/**`, `*.yml`, `Gemfile*`, …). Changes to `README.md` and most docs do not trigger a deploy. Workflows use Ruby 3.3, because 3.3.5/3.2.2 are unavailable on the ubuntu-26.04 runners. `bin/deploy` is the manual fallback and needs a global `purgecss`.

The `update-tocs.yml` workflow auto-commits regenerated `<!--ts-->…<!--te-->` blocks in Markdown files. Expect to pull that commit after editing headings in `README.md` or `docs/`.
