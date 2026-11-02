# Changelog

All notable changes to this project are documented in this file.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.4.0] - 2026-11-03

### Added
- **Cookbook (Recipes) integration** — new `Cookbook` module over the Nextcloud
  Cookbook REST API (`/index.php/apps/cookbook/api/v1`), matching the feature set
  and conventions of the existing Notes/Deck integrations. Requires the Cookbook
  app to be installed on the instance.
  - `recipes list [--query <q> | --category <c> | --keywords <k1,k2>]` — compact
    stubs (id, name, category, keywords, dates, thumbnail, web UI link) backed by
    the app's search/category/tag endpoints.
  - `recipes get --recipe <id>` — full schema.org Recipe JSON (ingredients,
    instructions, yields, times, nutrition, tools, image) plus synthesized
    `imageUrl` and Cookbook web UI `url`.
  - `recipes create` / `recipes edit --recipe <id>` — flags for title,
    description, category, ingredients, instructions, keywords, yield, tools,
    image, prep/cook/total time, and nine `--nutrition-*` fields. `edit` is
    read-modify-write: only the flags you pass change. Bulk list values split on
    `--`, then newlines, then `;`. Duration flags accept ISO 8601 (`PT1H30M`) or
    free text (`90`, `90 min`, `1h30`, `2 hours`).
  - `recipes delete --recipe <id>`, `recipes import --url <url> [--title <t>]`
    (server-side scraper import of public recipe pages), and
    `recipes image --recipe <id> [--size full|thumb|thumb16] [--out <path>]`.
  - `categories list` / `categories rename --category <c> --name <n>`;
    `keywords list`.
  - `cookbook config` (read) with `--folder`, `--interval`, `--print-image`,
    `--visible-blocks` updates; `cookbook reindex`.
- Cookbook write semantics verified against cookbook 0.11.x (API v1.2) and
  handled in the module: error bodies are plain-text/HTML and some success
  bodies are JSON-encoded scalars, so a dedicated `_api()` decoder replaces
  `request()`; 404 distinguishes "recipe not found" from "Cookbook app not
  installed"; PUT payloads strip server-injected presentation fields
  (`printImage`, thumb `imageUrl`, `@context`, `@type`) so the image is not
  re-downloaded on every save, and round-trip `dateCreated` so edits do not
  reset the creation timestamp.
- `tmp/cookbook-itest.sh` — live-instance integration script covering the full
  recipe/category/keyword/config/import lifecycle (67 assertions) and cleaning
  up after itself.
- Skill docs: Cookbook feature section, command reference, output formats,
  destructive-operation safety rows (`recipes:delete`, `categories:rename`,
  `cookbook:config`, `cookbook:reindex`), `default_recipe_category` agent
  memory behavior, recipe presentation guidance, and README usage examples.
- `CHANGELOG.md`.

## [0.3.0] - 2026-07-05

### Added
- **Deck (Kanban) integration** — `Deck` module over the Deck REST API
  (`/index.php/apps/deck/api/v1.1`) plus the OCS comments endpoint:
  `boards`, `stacks`, `cards`, and `labels` commands with list/get/create/edit/
  delete; `cards move`, `cards assign-label` / `remove-label`, and card
  comments (`cards comment-list` / `comment-add` / `comment-delete`).
- Card move uses the reorder endpoint with the *target* stack in the path;
  board deletion is a soft-delete, so `boards list` hides deleted boards.
- `flake.nix` / `flake.lock` providing the Node.js + esbuild toolchain so the
  bundle can be built via `nix shell`; `tmp/` is git-ignored.
- Skill docs for the Deck surface (commands, stack/card output formats,
  `default_deck_board` agent memory behavior, presentation guidance).

## [0.2.5] - 2026-05-15

### Added
- `DESCRIPTION` is exposed on calendar event and task reads (#7).

## [0.2.4] - 2026-05-02

### Fixed
- Secondary calendars declaring multiple component types are no longer missed
  during CalDAV discovery (#6).

## [0.2.3] - 2026-04-30

### Changed
- SKILL.md reworded to avoid tripping prompt-injection filters.

## [0.2.2] - 2026-04-30

### Changed
- Skill metadata aligned with the clawhub schema.

## [0.2.1] - 2026-04-30

### Added
- Agent-facing safety guidance in SKILL.md (confirm before destructive
  operations, treat retrieved content as untrusted).

## [0.2.0] - 2026-04-30

### Added
- File ids and synthesized `internalLink` on file listings/search results;
  `files upload` creates missing parent directories; public-link shares via
  `shares create-link` / `shares list` / `shares delete` (from the schemann fork).

### Changed
- Security hardening: HTTPS enforced for `NEXTCLOUD_URL` (opt-out via
  `OPENCLAW_ALLOW_HTTP=1`), token sent only to the configured instance.
- Flexible matching for `--calendar` / `--addressbook` (display name,
  case-insensitive name, URL slug, or full href).

## [0.1.x] - 2026-02-01

### Added
- Initial skill: Notes, Files (WebDAV), Calendar/Tasks (CalDAV), and Contacts
  (CardDAV) integrations behind a single bundled CLI.
