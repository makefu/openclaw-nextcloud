# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenClaw Nextcloud is a Node.js CLI skill that wraps Nextcloud APIs (Notes REST, WebDAV files, CalDAV calendars/tasks, CardDAV contacts, Deck REST, Cookbook REST) into a single command surface. It is distributed as a Claude Code Skill (see `SKILL.md`) that agents invoke via `Bash`.

## Build & Run

The project uses a **bundled architecture**: source lives in `index.js` and is bundled with all dependencies into `scripts/nextcloud.js` so end users/agents can run the skill with no `npm install`.

```bash
npm install               # only needed for development (esbuild)
npm run build             # bundles index.js -> scripts/nextcloud.js
node scripts/nextcloud.js <command> <subcommand> [options]
```

**Always commit both `index.js` and the rebuilt `scripts/nextcloud.js` together** — shipping only the source breaks the no-install promise that the skill depends on.

There is no test suite, linter, or typechecker configured. Verify changes by running `node scripts/nextcloud.js <cmd>` against a live Nextcloud instance with `NEXTCLOUD_URL`, `NEXTCLOUD_USER`, `NEXTCLOUD_TOKEN` set. `tmp/deck-itest.sh` and `tmp/cookbook-itest.sh` are live-instance integration scripts that exercise the full lifecycle of their service and clean up after themselves; run them after changing `Deck` or `Cookbook`.

## Architecture

`index.js` is a single-file CLI organized into one module object per service, all sharing one `request()` helper that handles Basic Auth and content-type-aware response parsing (JSON / XML via `fast-xml-parser` / text):

- **`Notes`** — Nextcloud Notes REST API at `/index.php/apps/notes/api/v1/notes`. Returns JSON directly.
- **`Files`** — WebDAV at `/remote.php/dav/files/{user}/`. Uses `PROPFIND` for listing, `SEARCH` (basicsearch XML) for queries, and `PUT`/`GET`/`DELETE` for file ops.
- **`CalDAV`** — Calendar events (VEVENT) and tasks (VTODO) at `/remote.php/dav/calendars/{user}/`. Calendars are discovered via `findCalendars(componentType)` which filters by `cal:supported-calendar-component-set`. Events/tasks are fetched via `REPORT` queries and **parsed from raw iCalendar text using regex** (not a vCalendar library) — same for updates via `_updateProperty()`.
- **`Contacts`** — CardDAV at `/remote.php/dav/addressbooks/users/{user}/`. vCards are likewise parsed and patched with regex (`_parseVCard`, `_updateVCardField`).
- **`Deck`** — Kanban boards/stacks/cards/labels via REST at `/index.php/apps/deck/api/v1.1`, plus card comments via the OCS endpoint. `PUT` replaces whole objects, so edits are read-modify-write merges onto `getBoard`/`getCard`.
- **`Cookbook`** — Recipes for the Cookbook app via REST at `/index.php/apps/cookbook/api/v1`. Recipes are schema.org Recipe JSON stored as `recipe.json` in a per-recipe folder. This module uses its own `_api()` wrapper instead of `request()` because the app returns plain-text/HTML error bodies and JSON-encoded scalars (e.g. `"Recipe 123 deleted successfully"`), and it must distinguish 404 "app not installed" from 404 "no such recipe". Writes are read-modify-write: `edit()` fetches the recipe, strips server-injected presentation fields (`_SERVER_FIELDS`: `printImage`, `imageUrl`, `@context`, `@type` — but NOT `dateCreated`/`dateModified`, which must round-trip or the creation timestamp resets), merges the passed flags, and sends the numeric `id` back so the server updates in place (and renames the folder).

The bottom of `index.js` is a flat `main()` switch over `command`/`subCommand` strings that pulls flag values out of `process.argv` by index. Every code path either calls `output()` (success JSON to stdout) or `errorOutput()` (error JSON to stderr + `exit(1)`).

### Conventions to preserve when extending

- All output is **machine-readable JSON** (`{status, data}` or `{status, message}`) — never log free text to stdout.
- New entities go in their own module object (like `Notes`, `Files`, etc.) and a new `else if (command === '...')` block in `main()`.
- For CalDAV/CardDAV writes: generate `crypto.randomUUID()` for the resource UID, `PUT` with `If-None-Match: *` for create and `If-Match: <etag>` for update.
- Calendar/address book selection follows a "find or default to first" pattern (`getCalendar`, `getAddressBook`); preserve this so the skill works without explicit `--calendar`/`--addressbook` flags.

## Skill Behavior (SKILL.md)

`SKILL.md` is the agent-facing contract — it is loaded by Claude Code when the skill is invoked. The behaviors below are load-bearing and should be respected by any agent working in this repo:

1. **Default selection memory** — the skill expects the agent to remember `default_task_calendar`, `default_event_calendar`, `default_addressbook`, `default_deck_board`, and `default_recipe_category` across invocations rather than asking each time.
2. **No-markdown output** — when *presenting* skill results to the user, use plain text + emojis (output may be relayed to Telegram/WhatsApp). This applies to user-facing summaries, not to the JSON the CLI itself emits.
3. **Cookbook requires the app** — `recipes` / `categories` / `keywords` / `cookbook` commands only work when the Nextcloud Cookbook app is installed; `_api()` raises a message naming that prerequisite on a 404.
