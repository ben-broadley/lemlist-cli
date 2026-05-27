# lemlist CLI

Read-only command-line wrapper over the [lemlist REST API](https://developer.lemlist.com/). Covers every GET endpoint (52 total) so you can pull a full snapshot of your lemlist tenant into JSON for analysis, auditing, or month-over-month diffing.

Stdlib-only Python 3 — no `pip install`, no dependencies.

Auto-generated from `openapi.json` (snapshot of `https://developer.lemlist.com/api-reference/openapi/v2.json`). To refresh: re-download that file and overwrite `openapi.json` — no code changes needed.

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
./lemlist commands                     # list every command
./lemlist help get-many-campaigns      # show args/flags for one command
./lemlist get-team                     # one-shot call → stdout JSON
./lemlist get-many-campaigns --status running --all
./lemlist get-campaign cam_xxx --out cam_xxx.json
./lemlist get-campaign-stats cam_xxx --startDate 2026-01-01 --endDate 2026-05-01
```

Path params are positional, query params are `--flags`. `--all` auto-paginates (offset/limit and page-based). `--out PATH` writes JSON to a file instead of stdout.

## Bulk dump

```
./lemlist dump-all dumps/2026-05-04
./lemlist dump-all dumps/2026-05-04 --with-campaigns
```

- **Phase 1**: every list endpoint with no required path arg → `<dir>/lists/<command>.json`
- **Phase 2** (`--with-campaigns`): for every campaign returned by `get-many-campaigns`, drills into per-campaign endpoints (`/campaigns/{id}`, `/sequences`, `/leads`, `/schedules`, `/statutes`, `/v2/campaigns/{id}/stats` — stats needs date flags so it'll skip).
- Endpoints requiring IDs/emails (e.g. `get-contact`, `get-lead-by-email`) are skipped and listed at the end — call them directly with the relevant ID.
- Endpoints requiring filters that have no obvious default (e.g. `get-many-inboxes` needs `--userId`, `get-campaign-reports` needs `--campaignIds`) are also reported as skipped.

Dumps are resumable — re-running against the same target dir skips files that already exist. Date-stamp the directory so you can diff snapshots over time.

## Auth detail

lemlist uses HTTP Basic with the API key as the password and an empty username. Override with `--user NAME` if needed.

## Files

- `lemlist` — the CLI (executable Python, stdlib-only)
- `openapi.json` — cached spec; the CLI builds its command table from this at runtime

## License

MIT.
