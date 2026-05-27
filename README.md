# lemlist CLI

An **unofficial** command-line wrapper over the [lemlist REST API](https://developer.lemlist.com/). Auto-generated from lemlist's published OpenAPI spec — every operation in the spec is a command (120 total: 52 GET, 45 POST, 8 PATCH, 1 PUT, 14 DELETE).

Stdlib-only Python 3 — no `pip install`, no dependencies.

> Not affiliated with, endorsed by, or sponsored by lemlist S.A.S. "lemlist" is a trademark of lemlist S.A.S.; use of the name here refers only to the API this tool talks to.

> **Heads up — this CLI can mutate your tenant.** POST/PUT/PATCH/DELETE commands write to lemlist. `dump-all` is read-only (GET only) — safe to run any time.

## Install

```
git clone https://github.com/ben-broadley/lemlist-cli.git
cd lemlist-cli
export LEMLIST_API_KEY=your_key_here    # from lemlist → Settings → Integrations → API
./lemlist commands
```

For repeat use, drop the key into a file you can `source`:

```
echo 'export LEMLIST_API_KEY=your_key_here' > ~/.lemlist.env
chmod 600 ~/.lemlist.env
source ~/.lemlist.env && ./lemlist get-team
```

Requires Python 3 (any recent version). No other dependencies.

## Quick start

```
./lemlist commands                     # list every command + HTTP method
./lemlist help create-campaign         # show args, body fields, required flags
./lemlist get-team                     # one-shot call → stdout JSON
./lemlist get-many-campaigns --status running --all
./lemlist get-campaign cam_xxx --out cam_xxx.json
./lemlist get-campaign-stats cam_xxx --startDate 2026-01-01 --endDate 2026-05-01
```

Path params are positional, query params are `--flags`. `--all` auto-paginates (offset/limit and page-based). `--out PATH` writes JSON to a file instead of stdout.

## Write methods

For POST / PUT / PATCH, the help output extracts the body schema from the spec. Top-level **primitive** fields (string/int/bool/number) are exposed as `--<field>` flags; complex fields (objects, arrays of objects) require `--body`:

```
# Simple body — primitive fields as flags
./lemlist create-campaign --name "Q3 outbound" --timezone Australia/Sydney

# Complex body — pass JSON directly
./lemlist update-campaign cam_xxx --body '{"sendUserIds":["usr_abc","usr_def"]}'

# Or load body from a file (or stdin)
./lemlist update-campaign cam_xxx --body-file body.json
./lemlist update-campaign cam_xxx --body-file -    < body.json

# Flags merge into --body, with flags winning on conflict
./lemlist update-campaign cam_xxx --body '{"name":"foo"}' --name "bar"
```

DELETE endpoints take only path params:

```
./lemlist delete-schedule sch_xxx
./lemlist delete-sequence-step seq_xxx stp_xxx
```

## Commands by domain

`./lemlist commands` prints the full alphabetised list. Below is the same surface area grouped by what each endpoint touches, so you can find the right command without scrolling 120 lines.

<details>
<summary><b>Campaigns</b> — create, update, pause, start, duplicate, stats, exports</summary>

```
get-many-campaigns                       GET    /campaigns
get-campaign                             GET    /campaigns/{campaignId}
get-campaign-reports                     GET    /campaigns/reports
get-campaign-sequences                   GET    /campaigns/{campaignId}/sequences
get-campaign-schedules                   GET    /campaigns/{campaignId}/schedules
get-campaign-statutes                    GET    /campaigns/{campaignId}/statutes
get-campaign-stats                       GET    /v2/campaigns/{campaignId}/stats
get-batch-campaign-stats                 POST   /v2/campaigns/stats/batch
create-campaign                          POST   /campaigns
update-campaign                          PATCH  /campaigns/{campaignId}
duplicate-campaign                       POST   /campaigns/{campaignId}/duplicate
start-campaign                           POST   /campaigns/{campaignId}/start
pause-campaign                           POST   /campaigns/{campaignId}/pause
associate-schedule-with-campaign         POST   /campaigns/{campaignId}/schedules/{scheduleId}
export-campaign-leads                    GET    /campaigns/{campaignId}/export/leads
start-campaign-stats-export              GET    /campaigns/{campaignId}/export/start
get-campaign-export-status               GET    /campaigns/{campaignId}/export/{exportId}/status
set-email-for-campaign-export-notification  PUT  /campaigns/{campaignId}/export/{exportId}/email/{email}
```
</details>

<details>
<summary><b>Leads</b> — add, update, enrich, pause, mark interested, unsubscribe</summary>

```
get-lead-by-email                        GET    /leads/{email}
get-lead-by-email-or-id                  GET    /leads
get-campaign-leads                       GET    /campaigns/{campaignId}/leads
create-lead-in-campaign                  POST   /campaigns/{campaignId}/leads
update-lead-in-a-campaign                PATCH  /campaigns/{campaignId}/leads/{leadId}
import-leads-from-crm                    POST   /campaigns/{campaignId}/leads/import
enrich-lead                              POST   /leads/{leadId}/enrich
pause-lead                               POST   /leads/pause/{leadId}
resume-paused-lead                       POST   /leads/start/{leadId}
mark-lead-as-interested                  POST   /leads/interested/{leadIdOrEmail}
mark-lead-as-not-interested              POST   /leads/notinterested/{leadIdOrEmail}
mark-lead-as-interested-in-campaign      POST   /campaigns/{campaignId}/leads/{leadIdOrEmail}/interested
mark-lead-as-not-interested-in-campaign  POST   /campaigns/{campaignId}/leads/{leadIdOrEmail}/notinterested
delete-or-unsubscribe-lead               DELETE /campaigns/{campaignId}/leads/{leadId}
unsubscribe-lead-from-campaign           DELETE /campaigns/{campaignId}/leads
add-custom-variables-on-leads            POST   /leads/{leadId}/variables
update-values-of-custom-variables-of-a-lead  PATCH  /leads/{leadId}/variables
erase-values-of-custom-variables-on-a-lead   DELETE /leads/{leadId}/variables
upload-audio-for-voice-message-step      POST   /leads/audio
```
</details>

<details>
<summary><b>Sequences</b> — add / update / delete steps</summary>

```
add-step-to-sequence                     POST   /sequences/{sequenceId}/steps
update-sequence-step                     PATCH  /sequences/{sequenceId}/steps/{stepId}
delete-sequence-step                     DELETE /sequences/{sequenceId}/steps/{stepId}
```
</details>

<details>
<summary><b>Schedules</b></summary>

```
get-many-schedules                       GET    /schedules
get-schedule                             GET    /schedules/{scheduleId}
create-schedule                          POST   /schedules
update-schedule                          PATCH  /schedules/{scheduleId}
delete-schedule                          DELETE /schedules/{scheduleId}
```
</details>

<details>
<summary><b>Contacts &amp; Contact Lists</b></summary>

```
get-many-contacts                        GET    /contacts
get-contact                              GET    /contacts/{idOrEmail}
add-and-update-contact                   POST   /contacts
get-contact-lists                        GET    /contacts/lists
create-contact-list                      POST   /contacts/lists
add-contacts-to-list                     POST   /contacts/lists/{listId}/entities
export-contact-list                      GET    /contacts/export
```
</details>

<details>
<summary><b>Companies</b></summary>

```
get-many-companies                       GET    /companies
add-and-update-company                   POST   /companies
get-company-notes                        GET    /companies/{companyId}/notes
create-company-note                      POST   /companies/{companyId}/notes
```
</details>

<details>
<summary><b>Inbox</b> — conversations, drafts, labels, send</summary>

```
get-many-inboxes                         GET    /inbox
get-contact-messages                     GET    /inbox/{contactId}
list-drafts                              GET    /inbox/{contactId}/drafts
get-draft                                GET    /inbox/{contactId}/drafts/{draftId}
create-draft                             POST   /inbox/{contactId}/drafts
update-draft                             PATCH  /inbox/{contactId}/drafts/{draftId}
delete-draft                             DELETE /inbox/{contactId}/drafts/{draftId}
send-email                               POST   /inbox/email
send-linkedin-message                    POST   /inbox/linkedin
send-whatsapp-message                    POST   /inbox/whatsapp
get-many-labels                          GET    /inbox/labels
get-label                                GET    /inbox/labels/{labelId}
create-label                             POST   /inbox/labels
attach-labels-to-conversations           POST   /inbox/conversations/labels/{contactId}
remove-labels-from-conversation          DELETE /inbox/conversations/labels/{contactId}
```
</details>

<details>
<summary><b>Lemwarm</b></summary>

```
get-lemwarm-settings                     GET    /lemwarm/{userMailboxId}/settings
update-lemwarm-settings                  PATCH  /lemwarm/{userMailboxId}/settings
start-lemwarm                            POST   /lemwarm/{userMailboxId}/start
pause-lemwarm                            POST   /lemwarm/{userMailboxId}/pause
```
</details>

<details>
<summary><b>Unsubscribes</b></summary>

```
get-many-unsubscribes                    GET    /unsubscribes
get-unsubscribe-by-email                 GET    /unsubscribes/{email}
add-unsubscribe-email-or-domain          POST   /unsubscribes/{email}
delete-unsubscribe-email                 DELETE /unsubscribes/{email}
export-unsubscribes                      GET    /unsubs/export
get-contact-subscription-status          GET    /v2/unsubscribes/contacts/{contactId}
unsubscribe-contact                      POST   /v2/unsubscribes/contacts/{contactId}
re-subscribe-contact                     DELETE /v2/unsubscribes/contacts/{contactId}
list-unsubscribed-variables              GET    /v2/unsubscribes/variables
get-unsubscribed-variable                GET    /v2/unsubscribes/variables/{value}
unsubscribe-variable                     POST   /v2/unsubscribes/variables/{value}
re-subscribe-variable                    DELETE /v2/unsubscribes/variables/{value}
bulk-unsubscribe-variables               POST   /v2/unsubscribes/variables
export-unsubscribed-contacts             GET    /v2/unsubscribes/exports/contacts
export-unsubscribed-variables            GET    /v2/unsubscribes/exports/variables
```
</details>

<details>
<summary><b>Webhooks</b></summary>

```
get-many-webhooks                        GET    /hooks
add-webhook                              POST   /hooks
delete-webhook                           DELETE /hooks/{hookId}
```
</details>

<details>
<summary><b>Team, Users, Email Accounts</b></summary>

```
get-team                                 GET    /team
get-team-credits                         GET    /team/credits
get-team-senders                         GET    /team/senders
get-team-crm-users                       GET    /team/crmUsers
get-user                                 GET    /users/{userId}
get-user-channels                        GET    /user/channels
connect-email-account                    POST   /user/email-accounts
disconnect-email-account                 DELETE /user/email-accounts/{emailAccountId}
test-email-account                       POST   /user/email-accounts/{emailAccountId}/test
```
</details>

<details>
<summary><b>Tasks</b></summary>

```
get-many-tasks                           GET    /tasks
create-task                              POST   /tasks
update-task                              PATCH  /tasks
ignore-tasks                             POST   /tasks/ignore
```
</details>

<details>
<summary><b>Activities</b></summary>

```
get-many-activities                      GET    /activities
delete-activity-recording-transcript     DELETE /activities/{activityId}/recording-transcript
```
</details>

<details>
<summary><b>Enrichment</b></summary>

```
enrich-data                              POST   /enrich
get-enrichment-result                    GET    /enrich/{enrichId}
bulk-enrich-data                         POST   /v2/enrichments/bulk
```
</details>

<details>
<summary><b>lemleads (B2B database)</b></summary>

```
search-people-database                   POST   /database/people
search-companies-database                POST   /database/companies
get-database-filters                     GET    /database/filters
get-people-schema                        GET    /schema/people
get-companies-schema                     GET    /schema/companies
```
</details>

<details>
<summary><b>Misc</b> — fields, CRM filters, watchlist</summary>

```
list-fields                              GET    /fields
get-crm-filters                          GET    /crm/filters
get-watchlist-signals                    GET    /watchlist/signals
```
</details>

## Bulk dump (read-only)

```
./lemlist dump-all dumps/2026-05-04
./lemlist dump-all dumps/2026-05-04 --with-campaigns
./lemlist dump-all dumps/2026-05-04 --deep   # everything (campaigns + users + lists + companies + schedules + search)
```

- **Phase 1**: every GET list endpoint with no required path arg → `<dir>/lists/<command>.json`
- **Phase 2** (`--with-campaigns`): for every campaign returned by `get-many-campaigns`, drills into per-campaign endpoints (`/campaigns/{id}`, `/sequences`, `/leads`, `/schedules`, `/statutes`).
- **Phase 3** (`--with-users`): per-user profile + inbox + per-mailbox lemwarm settings.
- **Phase 4** (`--with-lists`): exports the contents of every contact list.
- **Phase 5** (`--with-companies`): per-company notes.
- **Phase 6** (`--with-schedules`): full per-schedule details.
- **Phase 7** (`--with-search`): the read-only lemleads POST endpoints (`search-people-database`, `search-companies-database`) and a chunked 365-day `get-batch-campaign-stats` across every campaign.
- Endpoints requiring IDs/emails (e.g. `get-contact`, `get-lead-by-email`) are skipped and listed at the end — call them directly with the relevant ID.
- Write methods (POST/PUT/PATCH/DELETE) are **never invoked by dump-all** — it only hits GET (plus the three explicit read-only POSTs above).

Dumps are resumable — re-running against the same target dir skips files that already exist. Date-stamp the directory so you can diff snapshots over time.

## Updating the spec

The CLI builds its command table from `openapi.json` at startup. To pick up new lemlist endpoints, just refresh that file — no code changes needed:

```
curl -fsSL https://developer.lemlist.com/api-reference/openapi/v2.json -o openapi.json
./lemlist commands    # the new commands appear immediately
```

If lemlist removes or renames an endpoint, the corresponding command disappears (or gets a new slug derived from the new `summary`/`operationId`). Run `./lemlist commands` after a refresh to see the diff.

## Auth detail

lemlist uses HTTP Basic with the API key as the password and an empty username. Override with `--user NAME` if needed (rarely required).

## Troubleshooting

- **`HTTP 401`** — `LEMLIST_API_KEY` isn't set in your environment, or the key is wrong. Check with `echo $LEMLIST_API_KEY` and re-`source` your env file.
- **`HTTP 403`** — the key is valid but your lemlist plan doesn't include that endpoint (lemleads database search, enrichment, and some v2 endpoints are tier-gated).
- **`Malformed filters`** on `get-many-tasks` — observed on some tenants; lemlist's API quirk, not the CLI. Try `--status pending` or skip the endpoint.
- **`search-people-database` returns 0 results** — the default body sends `{"filters": [], "page": 1, "size": 100}`. lemlist's people DB requires at least one filter to return data. Discover valid filter IDs via `get-database-filters` and pass a real body:
  ```
  ./lemlist search-people-database --body '{"filters":[{"filterId":"country","in":["AU"],"out":[]}],"page":1,"size":100}'
  ```
- **Endpoint requires a flag I can't fill in** — `dump-all` reports these at the end as "skipped (required-flag)". Call them directly with `--<flag>` set.
- **Timeouts on large endpoints** — `get-many-activities`, `get-many-contacts`, and `get-many-companies` can hold thousands of records per page and time out on huge tenants. They're excluded from `dump-all` by default — include explicitly with `--include get-many-activities`.

## Bug reports

[github.com/ben-broadley/lemlist-cli/issues](https://github.com/ben-broadley/lemlist-cli/issues)

## Files

- `lemlist` — the CLI (executable Python, stdlib-only)
- `openapi.json` — cached spec; the CLI builds its command table from this at runtime

## License

MIT.
