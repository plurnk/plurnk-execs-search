> **ABSORBED into the plurnk platform monorepo** — code, history, and issues now live in [`plurnk/plurnk-service`](https://github.com/plurnk/plurnk-service) under `plurnk-execs-search/`. This repo is an archived snapshot.

# @plurnk/plurnk-execs-search

Web search runtime executor for [plurnk-service](https://github.com/plurnk/plurnk-service)'s `exec` scheme. Dispatches an `<<EXEC[search]:pie recipes:EXEC` op to a [SearXNG](https://docs.searxng.org/) instance and returns a compact digest of results (`title` / `url` / `snippet`).

The first non-subprocess `@plurnk/plurnk-execs-*` sibling, built on the [plurnk-execs](https://github.com/plurnk/plurnk-execs) framework.

## Runtime tags

Each tag maps to a SearXNG search category (`categories=`):

| Tag | Glyph | Category |
|---|---|---|
| `search` | 🔎 | general |
| `images` | 🖼 | images |
| `videos` | 🎬 | videos |
| `news` | 📰 | news |
| `map` | 🗺 | map |
| `music` | 🎵 | music |
| `it` | 💻 | it |
| `science` | 🔬 | science |
| `social` | 💬 | social media |
| `downloadable` | 📥 | files |

Engine, language, and time-range selection ride the query string via SearXNG's native `!bang` and `:lang` syntax (e.g. `<<EXEC[search]:!gh node streams:EXEC`). External bangs (`!!`) are refused — they redirect rather than return results.

## Configuration (environment)

Every tunable is an **optional env override** — no code default hides a magic number (suggested values live in the consuming service's `.env.example`).

| Var | Required | Behavior if unset |
|---|---|---|
| `PLURNK_EXECS_SEARCH_SEARXNG_URL` | **yes** | search is unavailable — base URL of the instance (`/search` must allow `format=json`) |
| `PLURNK_EXECS_SEARCH_LANGUAGE` | no | SearXNG's own default |
| `PLURNK_EXECS_SEARCH_LIMIT` | no | keep all results (else a client-side cap) |
| `PLURNK_EXECS_SEARCH_TIMEOUT` | no | the consumer's signal is the deadline (SPEC §2.5); this is an extra ceiling (ms) |
| `PLURNK_EXECS_SEARCH_SAFESEARCH` | no | instance default — `0` / `1` / `2` |
| `PLURNK_EXECS_SEARCH_SNIPPET` | no | snippet unbounded (else max chars per result snippet) |
| `PLURNK_EXECS_SEARCH_RAW` | no | digest mode; truthy → verbatim SearXNG payload, page pass skipped (debug) |
| `PLURNK_EXECS_SEARCH_PAGE_TIMEOUT` | no | exec signal is the deadline (else an extra per-page ceiling, ms) |
| `PLURNK_EXECS_SEARCH_REDIRECTS` | no | 3xx pruned (else max hops per page, each hop re-guarded) |

## The one-load flow (plurnk-execs#18)

Every candidate page is fetched **exactly once** (parallel, deduped by url):

- **Pruned, silently:** guard-refused targets (private/loopback/metadata/localhost — search results are attacker-influencable URLs, so every target and every redirect hop passes a public-address check), unreachable hosts, non-2xx, non-textual mimetypes, empty bodies. *Listed = loaded.*
- **Materialized:** each survivor becomes an `https://` entry via the consumer's `ExecArgs.entry()` sink, tagged with the slugified query (`pie_recipes`) — the consumer's ambience announces it as a folded row carrying path + tokens. Without the sink (older consumers) the flow degrades gracefully: prune + digest, no materialization.

## Output

Writes a compact **digest of survivors only** — `{ title, url, snippet }` per result (plus `publishedDate` when present), capped by `PLURNK_EXECS_SEARCH_LIMIT` — as JSON to the `results` channel. Zero dead rows by construction. The digest is the model's chooser context and rides OPEN (a few KB by design — the raw SearXNG payload was ~10–20× that and blew budgets, plurnk-execs#17); page bodies live in the materialized entries, never the packet. The model reads `exec://<coord>/EXEC#results`, then READs / `~`-queries the entries it picks.

Failures emit a `TelemetryEvent` (`source: "exec:<tag>"`): `searxng_not_configured`, `searxng_unreachable`, `searxng_timeout`, `searxng_http_<n>`, `external_bang_refused`.

## Tests

`test:lint`, `test:unit`.