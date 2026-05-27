# lemlist CLI

Command-line wrapper over the full [lemlist REST API](https://developer.lemlist.com/). Auto-generated from lemlist's published OpenAPI spec — every operation in the spec is a command (120 total: 52 GET, 45 POST, 8 PATCH, 1 PUT, 14 DELETE).

Stdlib-only Python 3 — no `pip install`, no dependencies.

To refresh: re-download `https://developer.lemlist.com/api-reference/openapi/v2.json` and overwrite `openapi.json` — no code changes needed.

> **Heads up — this CLI can mutate your tenant.** POST/PUT/PATCH/DELETE commands write to lemlist. `dump-all` is read-only (GET only) — safe to run any time.

## Setup

Get your API key from lemlist → Settings → Integrations → API.

```
git clone https://github.com/ben-broadley/lemlist-cli.git
cd lemlist-cli
export LEMLIST_API_KEY=your_key_here
./lemlist commands
```

For repeat use, drop the key into a file you can `source`:

```
echo 'export LEMLIST_API_KEY=your_key_here' > ~/.lemlist.env
chmod 600 ~/.lemlist.env
source ~/.lemlist.env && ./lemlist get-team
```

## Usage

```
./lemlist commands                     # list every command + HTTP method
./lemlist help create-campaign         # show args, body fields, required flags
./lemlist get-team                     # one-shot call → stdout JSON
./lemlist get-many-campaigns --status running --all
./lemlist get-campaign cam_xxx --out cam_xxx.json
./lemlist get-campaign-stats cam_xxx --startDate 2026-01-01 --endDate 2026-05-01
```

Path params are positional, query params are `--flags`. `--all` auto-paginates (offset/limit and page-based). `--out PATH` writes JSON to a file instead of stdout.

### Write methods

For POST / PUT / PATCH, the help output extracts the body schema from the spec. Top-level **primitive** fields (string/int/bool/number) are exposed as `--<field>` flags; complex fields (objects, arrays of objects) require `--body`:

```
# Simple body — primitive fields as flags
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

```
./lemlist delete-schedule sch_xxx
./lemlist delete-sequence-step seq_xxx stp_xxx
```

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

## Auth detail

lemlist uses HTTP Basic with the API key as the password and an empty username. Override with `--user NAME` if needed.

## Files

- `lemlist` — the CLI (executable Python, stdlib-only)
- `openapi.json` — cached spec; the CLI builds its command table from this at runtime

## License

MIT.
