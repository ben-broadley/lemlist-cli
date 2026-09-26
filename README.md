# lem

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3](https://img.shields.io/badge/python-3.x-blue.svg)](https://www.python.org/)

> One CLI over lemlist: the official REST CLI, lemlist's MCP action surface, and a resumable read-only tenant snapshot — behind a single entry point. Stdlib-only Python 3, no dependencies of its own.

> [!IMPORTANT]
> **Unofficial.** An independent open-source project, **not affiliated with, endorsed by, or sponsored by lemlist S.A.S.** "lemlist" is a trademark of lemlist S.A.S.; use of the name here refers only to the public API and MCP endpoint this tool talks to.

---

## Why this exists

lemlist ships two surfaces, and neither one covers the job on its own.

- **The official CLI** (`@lemlist-official/cli`) is good, and it beats what this repo used to be: it generates its commands from the *live* OpenAPI spec, so it never goes stale. **Use it. We delegate to it rather than racing it.**
- **The MCP server** is not a subset of the REST API — it's a different, larger surface. 124 tools, and a chunk of them have no REST endpoint at all: domain purchase, mailbox provisioning, deliverability audits, inbox placement tests, AI variable prompts, campaign folders, business context.

So you need both. But wiring up the MCP costs you **271 KB of tool schemas — roughly 65k tokens — loaded into an agent's context before it does anything.** `lem` drives the same MCP endpoint as plain commands: no MCP client, no schema tax, and the tools stay discoverable via `lem mcp tools`.

Plus the one thing that was always ours: `dump-all`, a resumable GET-only snapshot of a whole tenant.

## The three surfaces

| | what it is | backed by |
|---|---|---|
| `lem rest ...` | 143 REST operations | delegates to `@lemlist-official/cli` |
| `lem mcp ...` | 124 MCP actions | direct JSON-RPC to `app.lemlist.com/mcp` |
| `lem dump-all ...` | resumable read-only snapshot | live OpenAPI spec, direct HTTP |

---

## Installation

Requires **Python 3** and **Node.js 20+** (the latter for the official CLI we delegate to).

```bash
npm install -g @lemlist-official/cli
git clone https://github.com/ben-broadley/lemlist-cli.git
cd lemlist-cli
ln -s "$(pwd)/lem" ~/.local/bin/lem
```

Get your API key from lemlist (**Settings → Integrations → API**), then:

```bash
export LEMLIST_API_KEY=...
lem doctor
```

`lem doctor` checks all five moving parts and tells you which one is unhappy:

```
lem 0.3.0

[ok  ] python  — 3.14
[ok  ] official lemlist CLI  — /opt/homebrew/bin/lemlist (0.3.1)
[ok  ] LEMLIST_API_KEY  — set
[ok  ] live OpenAPI spec  — 101 paths / 143 operations
[ok  ] MCP endpoint  — 124 tools (59 read-only, 65 mutating)
```

---

## `lem rest` — the REST surface

Straight pass-through to the official CLI, with `--key` injected from `LEMLIST_API_KEY` so you don't have to think about it.

```bash
lem rest campaigns list
lem rest api GET /campaigns
lem rest --json team members | jq '.members[] | {email, role}'
```

Named profiles and OAuth still work — `lem rest auth login`, `lem rest --account client-x campaigns list`. We only inject the key when you haven't chosen a credential yourself.

---

## `lem mcp` — the MCP action surface

```bash
lem mcp tools                          # all 124, tagged read / write / DANGER
lem mcp tools --reads                  # just the safe ones
lem mcp tools --search deliverability  # filter by name or description
lem mcp help check_domain_health       # schema, arguments, safety class
lem mcp call get_team_info
lem mcp call check_domain_health --set scope=team
```

Arguments go in via `--set k=v` (values are parsed as JSON when they parse, so `--set ids='["a","b"]'` and `--set limit=10` do what you want), or as a whole object via `--data '{...}'`.

The tool list is cached for 24h in `~/.lem/`; `--refresh` re-reads it, so tools lemlist ships tomorrow show up without a release here.

### Coverage

**All 124 tools, no allowlist.** The command surface *is* `tools/list` — there are no hand-written wrappers to fall behind. Verified against a live tenant: 124/124 resolve and render their schema, and 32/32 of the read-only zero-argument set execute and return real data.

Two caveats worth knowing:

- **The schemas under-declare.** Several tools ship `required: []` but enforce an either/or rule server-side — `get_campaign_details` wants exactly one of `campaignId` or `campaignIds`. Nothing in the schema says so, so `lem` can't pre-validate it. The server's rejection is relayed verbatim and does explain the fix.
- **Workspace tools degrade headless.** `display_leads`, `display_table` and `propose_sequence` are meant to render into lemlist's UI panel. They return successfully from the CLI, but you get a data reference rather than a rendered table.

### Write guards

The MCP is 65 mutating tools deep, and some of them spend money. `lem` grades them:

- **read** — runs as-is.
- **write** — refuses without `--yes`.
- **DANGER** — refuses without **both** `--yes` and `--force-danger`. Anything that charges Stripe (`purchase_domain`), rewrites DNS, provisions mailboxes, sends real outbound (`send_message`, `launch_campaign`), or destroys data (every `delete_*`).

```bash
$ lem mcp call purchase_domain --set domain=acme.com --yes
purchase_domain is billing-affecting or irreversible.
  it needs BOTH --yes and --force-danger.
  inspect it first:  lem mcp help purchase_domain
```

`--dry-run` prints the exact request and sends nothing.

---

## `lem dump-all` — read-only tenant snapshot

GET-only, resumable, atomic writes. It cannot mutate your tenant.

```bash
lem dump-all ../lemlist-data                    # Phase 1: every list endpoint
lem dump-all ../lemlist-data --with-campaigns   # + per-campaign drill
lem dump-all ../lemlist-data --deep             # every phase
```

| flag | phase |
|---|---|
| *(none)* | 1 — list endpoints |
| `--with-campaigns` | 2 — per-campaign endpoints |
| `--with-users` | 3 — per-user profile, inbox, per-mailbox lemwarm |
| `--with-lists` | 4 — contact-list contents |
| `--with-companies` | 5 — per-company notes |
| `--with-schedules` | 6 — full schedule details |
| `--with-search` | 7 — the two read-only POST searches + batch campaign stats |
| `--deep` | all of the above |

Re-runs skip files that already exist, so an interrupted dump resumes where it stopped. `--force` re-fetches. Writes are staged through a `.tmp` and renamed, so a Ctrl+C leaves either complete JSON or nothing.

Some endpoints are excluded by default (mass collections, async export kickoffs, low-signal config) — the run prints exactly what it skipped and why. Pull one back in with `--include <name>`.

**Exclusions are keyed on `(METHOD, path)`, not on command names.** Names are slugified from the spec's `summary` field, and summaries churn: three of ours were silently renamed between the May 2026 spec and the live one. Paths are stable. This is why the snapshot survives spec revisions.

---

## Conventions

- JSON to stdout, progress and diagnostics to stderr — pipe freely.
- The `Authorization` / `X-API-Key` header is never logged, never printed in `--dry-run` output.
- Exit codes: `0` ok, `1` validation or refused guard, `5` not found, `6` MCP error, `127` official CLI missing.
- Cache lives in `~/.lem/` (override with `LEM_CACHE_DIR`). OpenAPI spec: 1h TTL. MCP tools: 24h.
- Point at a different official CLI binary with `LEMLIST_BIN`.

---

## Upgrading from v0.2

v0.2 was 120 REST commands generated from a **vendored** `openapi.json`. By August 2026 that snapshot was 12 paths and 23 operations behind the live API, and rotting further every week. The official CLI solves that properly, so v0.3 deletes our copy rather than maintaining a worse one.

| v0.2 | v0.3 |
|---|---|
| `lemlist get-many-campaigns` | `lem rest campaigns list` |
| `lemlist get-campaign cam_123` | `lem rest campaigns get cam_123` |
| `lemlist <any generated command>` | `lem rest api <METHOD> <path>` |
| `lemlist dump-all <dir>` | `lem dump-all <dir>` *(unchanged)* |
| — | `lem mcp ...` *(new)* |

The `lemlist` entry point still exists as a shim: it forwards `dump-all`, `doctor`, `rest` and `mcp`, and points anything else at its replacement. The vendored `openapi.json` is gone — the spec is fetched live and cached.

---

## Troubleshooting

**`official lemlist CLI not found on PATH`** — `npm install -g @lemlist-official/cli`, or set `LEMLIST_BIN` to its location.

**`set LEMLIST_API_KEY in your environment`** — export it, or run `lem rest auth login` to use an OAuth profile for the REST surface. Note that `lem mcp` and `lem dump-all` need the raw key; they don't read the official CLI's profile store.

**`MCP error -32602: X-API-Key header or Authorization: Bearer token is required`** — the key isn't reaching the MCP endpoint. Check `lem doctor`.

**A `dump-all` endpoint fails with a 400 about a required filter** — it's a lookup endpoint the spec marks as unfiltered. Call it directly through `lem rest` with the filter it wants, and open an issue so it can be added to `LOOKUP_ONLY_PATHS`.

**`lemlist: PATH loop`** — something earlier on `PATH` is also named `lemlist` and execs this repo's shim, so `lem rest` resolved back to itself. `lem` prefers a binary whose realpath sits inside `@lemlist-official` and refuses to call its own shim, but a wrapper that hides the real one still wins. Either point `LEMLIST_BIN` at the official CLI, or rename the wrapper so it stops claiming the name:

```bash
export LEMLIST_BIN=$(npm root -g)/@lemlist-official/cli/dist/cli.js
```

**Stale command surface** — `lem mcp tools --refresh` re-reads the tool list; `lem dump-all --refresh-spec` re-reads the OpenAPI spec.

---

## Bug reports

[github.com/ben-broadley/lemlist-cli/issues](https://github.com/ben-broadley/lemlist-cli/issues)

## License

MIT — see [LICENSE](LICENSE).
