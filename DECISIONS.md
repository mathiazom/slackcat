# Decisions

Current architecture/scope decisions for slackcat — kept short. Only what's
true now; when a decision changes, edit or delete the entry rather than keeping the
old one around. This is not a log of how we got here, just what to assume before
starting new work. Point at the source file instead of restating what's derivable
from the code (schema, API shapes, config values).

None of this is final. Every entry here is a starting assumption, not a ruling —
if new work gives a real reason to reconsider one, question it and change the entry
rather than working around it or treating it as settled.

## Rendering & auth

slackopy is SSR: each page checks the session (`getSession`) and branches to its
content or `Login`, querying slackback per request. The target end-state is fully
static generation with a thin Astro middleware auth-gate in front of it, but that
conversion happens in one future pass, not split across two, so the site is never
left ungated in between.

## slackpack: file schema and pack-time resolution

The `file` table is unversioned (Slack file IDs are immutable, unlike channel/user
snapshots). slackpack mutates a message's JSON at pack time to embed the archived
file pointer directly (`internal/packing/files.go`) — slackback and slackopy never
need to join against `file` themselves.

## Migrations are forward-only

One `.sql` file per migration, no down-migrations. Rollback is a Postgres
backup/restore, not a scripted reversal — down-scripts only ever safely reverse the
low-risk changes anyway, and this archive can't afford to trust an untested one
against irreplaceable data.

## Production setup is a generated quickstart, not hand-assembled files

`quickstart.sh` (repo root) generates the whole system as one deployment directory —
`docker-compose.yml`, `.env`, `fusionauth-kickstart.json` — instead of hand-assembled
files with no record of how they got there. Re-running preserves existing secrets/
values and rewrites the templates. Missing admin-only values (Slack credentials,
domains, reverse-proxy network name) are prompted for interactively, else left
blank and listed at the end. No bundled proxy or published ports — the deployment
joins the admin's existing proxy network instead (`REVERSE_PROXY_NETWORK`), since
that's the standard direction for these setups.