# lemlist CLI

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3](https://img.shields.io/badge/python-3.x-blue.svg)](https://www.python.org/)

> A stdlib-only Python CLI for the [lemlist](https://lemlist.com) REST API. Auto-generated from the published OpenAPI spec — covers every operation lemlist documents (120 commands across GET / POST / PUT / PATCH / DELETE).

> [!IMPORTANT]
> **Unofficial.** This is an independent open-source project and is **not affiliated with, endorsed by, or sponsored by lemlist S.A.S.** "lemlist" is a trademark of lemlist S.A.S.; use of the name here refers only to the public API this tool talks to.

> [!WARNING]
> This CLI can mutate your tenant. POST / PUT / PATCH / DELETE commands write to lemlist. The `dump-all` subcommand is read-only (GET only) — safe to run any time.

---

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Authentication](#authentication)
- [Output Conventions](#output-conventions)
- [Commands by Domain](#commands-by-domain)
- [Write Methods](#write-methods)
- [Bulk Dump (read-only)](#bulk-dump-read-only)
- [Global Options](#global-options)
- [Exit Codes](#exit-codes)
- [Updating the Spec](#updating-the-spec)
- [Troubleshooting](#troubleshooting)
- [Development](#development)
- [Bug Reports](#bug-reports)
- [License](#license)

---

## Installation

Requires **Python 3** (any recent version). Zero external dependencies.

```bash
git clone https://github.com/ben-broadley/lemlist-cli.git
cd lemlist-cli
./lemlist commands
```

Optionally put it on your `PATH`:

```bash
ln -s "$(pwd)/lemlist" ~/.local/bin/lemlist
```

---

## Quick Start

### 1. Get your API key

In lemlist, go to **Settings → Integrations → API** and copy your key.

### 2. Export it

```bash
export LEMLIST_API_KEY="your_key_here"
```

For repeat use, drop it into a `0600` file you can `source`:

```bash
echo 'export LEMLIST_API_KEY="your_key_here"' > ~/.lemlist.env
chmod 600 ~/.lemlist.env
source ~/.lemlist.env
```

### 3. Verify

```bash
$ ./lemlist get-team
{
  "_id": "tea_xxxxxxxxxxxxxxxxx",
  "name": "Acme Outreach",
  "userIds": ["usr_aaa", "usr_bbb", ...],
  "settings": { ... }
}
```

### 4. Explore

```bash
./lemlist commands                          # list every command + HTTP method
./lemlist help get-many-campaigns           # show args/flags for one command
./lemlist get-many-campaigns --status running --all
./lemlist get-campaign cam_xxx --out cam_xxx.json
```

### 5. Snapshot your whole tenant

```bash
./lemlist dump-all dumps/2026-05-04 --deep
```

---

## Authentication

The CLI resolves your API key in this order:

| Priority | Method                            | Example                                              |
| -------- | --------------------------------- | ---------------------------------------------------- |
| 1        | `LEMLIST_API_KEY` environment var | `export LEMLIST_API_KEY=xxx && ./lemlist get-team`   |
| 2        | Sourced env file                  | `source ~/.lemlist.env && ./lemlist get-team`        |

lemlist uses **HTTP Basic auth** with the API key as the password and an empty username. The CLI handles this transparently. Override the username with `--user NAME` if needed (rarely required).

---

## Output Conventions

| Stream | Goes to | Contents                                                              |
| ------ | ------- | --------------------------------------------------------------------- |
| stdout | data    | The JSON response from lemlist (or whatever `--out` writes to a file) |
| stderr | meta    | Progress messages (`dump-all` phases), warnings, error messages       |

This means piping and redirection always work cleanly:

```bash
./lemlist get-many-campaigns --all | jq '.[] | {id: ._id, name}'
./lemlist get-team > team.json 2> /dev/null
./lemlist dump-all dumps/2026-05-04 2> dump.log
```

All responses are pretty-printed JSON. Use `--out PATH` to write to a file instead of stdout (handy for very large payloads).

---

## Commands by Domain

`./lemlist commands` prints the full alphabetised list (120 entries). Grouped below by what each endpoint touches:

<details>
<summary><b>Campaigns</b> — create, update, pause, start, duplicate, stats, exports</summary>

```
get-many-campaigns                          GET    /campaigns
get-campaign                                GET    /campaigns/{campaignId}
get-campaign-reports                        GET    /campaigns/reports
get-campaign-sequences                      GET    /campaigns/{campaignId}/sequences
get-campaign-schedules                      GET    /campaigns/{campaignId}/schedules
get-campaign-statutes                       GET    /campaigns/{campaignId}/statutes
get-campaign-stats                          GET    /v2/campaigns/{campaignId}/stats
get-batch-campaign-stats                    POST   /v2/campaigns/stats/batch
create-campaign                             POST   /campaigns
update-campaign                             PATCH  /campaigns/{campaignId}
duplicate-campaign                          POST   /campaigns/{campaignId}/duplicate
start-campaign                              POST   /campaigns/{campaignId}/start
pause-campaign                              POST   /campaigns/{campaignId}/pause
associate-schedule-with-campaign            POST   /campaigns/{campaignId}/schedules/{scheduleId}
export-campaign-leads                       GET    /campaigns/{campaignId}/export/leads
start-campaign-stats-export                 GET    /campaigns/{campaignId}/export/start
get-campaign-export-status                  GET    /campaigns/{campaignId}/export/{exportId}/status
set-email-for-campaign-export-notification  PUT    /campaigns/{campaignId}/export/{exportId}/email/{email}
```
</details>

<details>
<summary><b>Leads</b> — add, update, enrich, pause, mark interested, unsubscribe</summary>

```
get-lead-by-email                            GET    /leads/{email}
get-lead-by-email-or-id                      GET    /leads
get-campaign-leads                           GET    /campaigns/{campaignId}/leads
create-lead-in-campaign                      POST   /campaigns/{campaignId}/leads
update-lead-in-a-campaign                    PATCH  /campaigns/{campaignId}/leads/{leadId}
import-leads-from-crm                        POST   /campaigns/{campaignId}/leads/import
enrich-lead                                  POST   /leads/{leadId}/enrich
pause-lead                                   POST   /leads/pause/{leadId}
resume-paused-lead                           POST   /leads/start/{leadId}
mark-lead-as-interested                      POST   /leads/interested/{leadIdOrEmail}
mark-lead-as-not-interested                  POST   /leads/notinterested/{leadIdOrEmail}
mark-lead-as-interested-in-campaign          POST   /campaigns/{campaignId}/leads/{leadIdOrEmail}/interested
mark-lead-as-not-interested-in-campaign      POST   /campaigns/{campaignId}/leads/{leadIdOrEmail}/notinterested
delete-or-unsubscribe-lead                   DELETE /campaigns/{campaignId}/leads/{leadId}
unsubscribe-lead-from-campaign               DELETE /campaigns/{campaignId}/leads
add-custom-variables-on-leads                POST   /leads/{leadId}/variables
update-values-of-custom-variables-of-a-lead  PATCH  /leads/{leadId}/variables
erase-values-of-custom-variables-on-a-lead   DELETE /leads/{leadId}/variables
upload-audio-for-voice-message-step          POST   /leads/audio
```
</details>

<details>
<summary><b>Sequences</b> — add / update / delete steps</summary>

```
add-step-to-sequence                        POST   /sequences/{sequenceId}/steps
update-sequence-step                        PATCH  /sequences/{sequenceId}/steps/{stepId}
delete-sequence-step                        DELETE /sequences/{sequenceId}/steps/{stepId}
```
</details>

<details>
<summary><b>Schedules</b></summary>

```
get-many-schedules                          GET    /schedules
get-schedule                                GET    /schedules/{scheduleId}
create-schedule                             POST   /schedules
update-schedule                             PATCH  /schedules/{scheduleId}
delete-schedule                             DELETE /schedules/{scheduleId}
```
</details>

<details>
<summary><b>Contacts &amp; Contact Lists</b></summary>

```
get-many-contacts                           GET    /contacts
get-contact                                 GET    /contacts/{idOrEmail}
add-and-update-contact                      POST   /contacts
get-contact-lists                           GET    /contacts/lists
create-contact-list                         POST   /contacts/lists
add-contacts-to-list                        POST   /contacts/lists/{listId}/entities
export-contact-list                         GET    /contacts/export
```
</details>

<details>
<summary><b>Companies</b></summary>

```
get-many-companies                          GET    /companies
add-and-update-company                      POST   /companies
get-company-notes                           GET    /companies/{companyId}/notes
create-company-note                         POST   /companies/{companyId}/notes
```
</details>

<details>
<summary><b>Inbox</b> — conversations, drafts, labels, send</summary>

```
get-many-inboxes                            GET    /inbox
get-contact-messages                        GET    /inbox/{contactId}
list-drafts                                 GET    /inbox/{contactId}/drafts
get-draft                                   GET    /inbox/{contactId}/drafts/{draftId}
create-draft                                POST   /inbox/{contactId}/drafts
update-draft                                PATCH  /inbox/{contactId}/drafts/{draftId}
delete-draft                                DELETE /inbox/{contactId}/drafts/{draftId}
send-email                                  POST   /inbox/email
send-linkedin-message                       POST   /inbox/linkedin
send-whatsapp-message                       POST   /inbox/whatsapp
get-many-labels                             GET    /inbox/labels
get-label                                   GET    /inbox/labels/{labelId}
create-label                                POST   /inbox/labels
attach-labels-to-conversations              POST   /inbox/conversations/labels/{contactId}
remove-labels-from-conversation             DELETE /inbox/conversations/labels/{contactId}
```
</details>

<details>
<summary><b>Lemwarm</b></summary>

```
get-lemwarm-settings                        GET    /lemwarm/{userMailboxId}/settings
update-lemwarm-settings                     PATCH  /lemwarm/{userMailboxId}/settings
start-lemwarm                               POST   /lemwarm/{userMailboxId}/start
pause-lemwarm                               POST   /lemwarm/{userMailboxId}/pause
```
</details>

<details>
<summary><b>Unsubscribes</b></summary>

```
get-many-unsubscribes                       GET    /unsubscribes
get-unsubscribe-by-email                    GET    /unsubscribes/{email}
add-unsubscribe-email-or-domain             POST   /unsubscribes/{email}
delete-unsubscribe-email                    DELETE /unsubscribes/{email}
export-unsubscribes                         GET    /unsubs/export
get-contact-subscription-status             GET    /v2/unsubscribes/contacts/{contactId}
unsubscribe-contact                         POST   /v2/unsubscribes/contacts/{contactId}
re-subscribe-contact                        DELETE /v2/unsubscribes/contacts/{contactId}
list-unsubscribed-variables                 GET    /v2/unsubscribes/variables
get-unsubscribed-variable                   GET    /v2/unsubscribes/variables/{value}
unsubscribe-variable                        POST   /v2/unsubscribes/variables/{value}
re-subscribe-variable                       DELETE /v2/unsubscribes/variables/{value}
bulk-unsubscribe-variables                  POST   /v2/unsubscribes/variables
export-unsubscribed-contacts                GET    /v2/unsubscribes/exports/contacts
export-unsubscribed-variables               GET    /v2/unsubscribes/exports/variables
```
</details>

<details>
<summary><b>Webhooks</b></summary>

```
get-many-webhooks                           GET    /hooks
add-webhook                                 POST   /hooks
delete-webhook                              DELETE /hooks/{hookId}
```
</details>

<details>
<summary><b>Team, Users, Email Accounts</b></summary>

```
get-team                                    GET    /team
get-team-credits                            GET    /team/credits
get-team-senders                            GET    /team/senders
get-team-crm-users                          GET    /team/crmUsers
get-user                                    GET    /users/{userId}
get-user-channels                           GET    /user/channels
connect-email-account                       POST   /user/email-accounts
disconnect-email-account                    DELETE /user/email-accounts/{emailAccountId}
test-email-account                          POST   /user/email-accounts/{emailAccountId}/test
```
</details>

<details>
<summary><b>Tasks</b></summary>

```
get-many-tasks                              GET    /tasks
create-task                                 POST   /tasks
update-task                                 PATCH  /tasks
ignore-tasks                                POST   /tasks/ignore
```
</details>

<details>
<summary><b>Activities</b></summary>

```
get-many-activities                         GET    /activities
delete-activity-recording-transcript        DELETE /activities/{activityId}/recording-transcript
```
</details>

<details>
<summary><b>Enrichment</b></summary>

```
enrich-data                                 POST   /enrich
get-enrichment-result                       GET    /enrich/{enrichId}
bulk-enrich-data                            POST   /v2/enrichments/bulk
```
</details>

<details>
<summary><b>lemleads (B2B database)</b></summary>

```
search-people-database                      POST   /database/people
search-companies-database                   POST   /database/companies
get-database-filters                        GET    /database/filters
get-people-schema                           GET    /schema/people
get-companies-schema                        GET    /schema/companies
```
</details>

<details>
<summary><b>Misc</b> — fields, CRM filters, watchlist</summary>

```
list-fields                                 GET    /fields
get-crm-filters                             GET    /crm/filters
get-watchlist-signals                       GET    /watchlist/signals
```
</details>

---

## Write Methods

For POST / PUT / PATCH, the help output extracts the body schema from the spec. Top-level **primitive** fields (string / integer / number / boolean) are exposed as `--<field>` flags. Complex fields (objects, arrays of objects) require `--body`.

```bash
# `help` shows what's available
$ ./lemlist help create-campaign
create-campaign
  POST /campaigns
  Create Campaign

Body flags:
  --body JSON     JSON string for the request body
  --body-file F   path to a JSON body file (use - for stdin)

  Top-level fields from the spec:
    --name (required)  [string] The name of the campaign
    --timezone         [string] IANA timezone for the campaign schedule
                                 (defaults to `Europe/Paris` if omitted)
```

```bash
# Simple body — use primitive flags
./lemlist create-campaign --name "Q3 outbound" --timezone Australia/Sydney

# Complex body — pass JSON directly
./lemlist update-campaign cam_xxx --body '{"sendUserIds":["usr_abc","usr_def"]}'

# Or load body from file (or stdin)
./lemlist update-campaign cam_xxx --body-file body.json
./lemlist update-campaign cam_xxx --body-file -    < body.json

# Flags merge into --body, with flags winning on conflict
./lemlist update-campaign cam_xxx --body '{"name":"foo"}' --name "bar"
```

DELETE endpoints take only path params:

```bash
./lemlist delete-schedule sch_xxx
./lemlist delete-sequence-step seq_xxx stp_xxx
```

---

## Bulk Dump (read-only)

```bash
./lemlist dump-all dumps/2026-05-04
./lemlist dump-all dumps/2026-05-04 --with-campaigns
./lemlist dump-all dumps/2026-05-04 --deep   # everything
```

`dump-all` snapshots your tenant by walking lemlist's list endpoints. It is **strictly read-only** — write methods (POST/PUT/PATCH/DELETE) are never invoked, and the three POSTs it does call (`search-people-database`, `search-companies-database`, `get-batch-campaign-stats`) are read-only search/aggregate endpoints.

| Flag                 | What it adds                                                                                                                                                |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| *(no flag)*          | **Phase 1**: every GET list endpoint with no required path arg → `<dir>/lists/<command>.json`                                                                |
| `--with-campaigns`   | **Phase 2**: per-campaign endpoints (`get-campaign`, `get-campaign-sequences`, `get-campaign-statutes`, ...) for every campaign returned by `get-many-campaigns` |
| `--with-users`       | **Phase 3**: per-user profile + inbox + per-mailbox lemwarm settings                                                                                         |
| `--with-lists`       | **Phase 4**: exports the contents of every contact list                                                                                                      |
| `--with-companies`   | **Phase 5**: per-company notes                                                                                                                              |
| `--with-schedules`   | **Phase 6**: full per-schedule details                                                                                                                      |
| `--with-search`      | **Phase 7**: read-only POSTs — lemleads search + chunked 365-day batch campaign stats                                                                       |
| `--deep`             | All of the above                                                                                                                                            |
| `--include name,...` | Force-include endpoints that are excluded by default (e.g. `get-many-activities`)                                                                           |
| `--force`            | Re-fetch endpoints whose output file already exists                                                                                                         |

Dumps are **resumable** — re-running against the same target directory skips files that already exist. Date-stamp the directory so you can diff snapshots over time.

```bash
# A monthly snapshot you can diff
./lemlist dump-all dumps/$(date +%Y-%m-%d) --deep

# A second run picks up where the first left off
./lemlist dump-all dumps/2026-05-04 --deep    # → "cached" for everything already pulled
```

Endpoints requiring IDs/emails (`get-contact`, `get-lead-by-email`) and large/low-signal collections (`get-many-activities`, `get-many-contacts`) are skipped by default and listed at the end of the run.

---

## Global Options

These flags work on every endpoint command:

| Flag               | Purpose                                                                                       |
| ------------------ | --------------------------------------------------------------------------------------------- |
| `--all`            | Auto-paginate (offset/limit or page-based). GET only.                                         |
| `--out PATH`       | Write JSON to a file instead of stdout. Prints byte count to stderr.                          |
| `--user NAME`      | Basic-auth username (default: empty — lemlist's standard).                                    |
| `--body JSON`      | JSON string for POST/PUT/PATCH body.                                                          |
| `--body-file PATH` | Load body from a file. Use `-` for stdin.                                                     |

For POST/PUT/PATCH, the script also adds `--<field>` flags for every primitive top-level body property in the spec (see `./lemlist help <command>`).

---

## Exit Codes

| Code | Meaning                                                                                            |
| ---- | -------------------------------------------------------------------------------------------------- |
| `0`  | Success.                                                                                            |
| `1`  | Any error: missing/wrong API key, missing required flag, network failure, lemlist returned 4xx/5xx, etc. The error message goes to stderr. |

(The CLI doesn't currently distinguish error types via exit codes; if you need to differentiate, check the stderr message or the script's stderr line for `HTTP <code>`.)

---

## Updating the Spec

The CLI builds its command table from `openapi.json` at startup. To pick up new lemlist endpoints, refresh that file — **no code changes needed**:

```bash
curl -fsSL https://developer.lemlist.com/api-reference/openapi/v2.json -o openapi.json
./lemlist commands    # new endpoints appear immediately
```

If lemlist removes or renames an endpoint, the corresponding command disappears (or gets a new slug derived from the new `summary`/`operationId`). Run `./lemlist commands` after a refresh to see the diff.

---

## Troubleshooting

| Symptom                                         | Likely cause / fix                                                                                                                                                                                                                                                                                              |
| ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `HTTP 401`                                      | `LEMLIST_API_KEY` isn't set, or the value is wrong. Check `echo $LEMLIST_API_KEY` and re-`source` your env file.                                                                                                                                                                                                |
| `HTTP 403`                                      | Key valid, but your lemlist plan doesn't include that endpoint (lemleads database search, enrichment, and some v2 endpoints are tier-gated).                                                                                                                                                                    |
| `Malformed filters` on `get-many-tasks`         | Observed on some tenants — lemlist API quirk, not the CLI. Pass `--status pending` (or another filter) to dodge it, or skip the endpoint.                                                                                                                                                                       |
| `search-people-database` returns 0 results      | The default body sends `{"filters": [], "page": 1, "size": 100}`. lemlist's people DB needs at least one filter to return data: `./lemlist search-people-database --body '{"filters":[{"filterId":"country","in":["AU"],"out":[]}],"page":1,"size":100}'`. Discover valid filter IDs via `get-database-filters`. |
| `dump-all` reports `skipped (required-flag)`    | The endpoint needs a flag the dump can't auto-fill. Call it directly with the right `--flag`.                                                                                                                                                                                                                   |
| Timeouts on `get-many-activities` / `-contacts` | These can hold thousands of records per page and time out on huge tenants. Excluded from `dump-all` by default — include explicitly: `--include get-many-activities`.                                                                                                                                           |

---

## Development

No build step — `lemlist` is a single executable Python file. To work on it:

```bash
git clone https://github.com/ben-broadley/lemlist-cli.git
cd lemlist-cli
# Edit `lemlist`, run it.
```

The command table is auto-built from `openapi.json` on every invocation. To pull a fresh spec from lemlist:

```bash
curl -fsSL https://developer.lemlist.com/api-reference/openapi/v2.json -o openapi.json
```

There are no unit tests at the moment — the OpenAPI spec is the contract. A quick sanity check:

```bash
./lemlist commands            # should list 120 commands
./lemlist help get-team       # should render without error
python3 -c "import ast; ast.parse(open('lemlist').read()); print('parses OK')"
```

---

## Bug Reports

[github.com/ben-broadley/lemlist-cli/issues](https://github.com/ben-broadley/lemlist-cli/issues)

---

## License

[MIT](LICENSE).
