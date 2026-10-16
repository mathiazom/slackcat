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
## slackback: channel views show the latest snapshot

`channel` rows are snapshots (one per detected metadata change), but the channel
list and channel page both show only the most recent snapshot per `public_id`.
Older snapshots are kept for a possible future historical timeline, but nothing
exposes them yet. Channels that are archived or deleted in Slack stay listed,
since this is an archive.

## slackback: users resolve as of message time

Authors, mentions and reactors use the user snapshot valid at the message ts, from
the `user` table only (not the `user_profile` Slack embeds in messages). Snapshots
are daily, so a profile change shows up to a day late; accepted. Before a user's
first snapshot: earliest name. Avatars resolve separately, by when the photo became
current (see avatars below), so a photo can change before the names do and show up
in pre-archive history. Before that: placeholder (we don't know their face then).
Message grouping still breaks on snapshot only, not on photo changes.

## slackpack: avatars

Archived per user snapshot into the unversioned `avatar` table, keyed by Slack URL,
with pointers embedded in the snapshot JSON at pack time like files
(`internal/packing/avatars.go`). Change detection ignores those pointers. Gone
images (403/404/410) are never retried. Bot avatars: not yet.

Each snapshot also gets `archived_avatar_since` (unix seconds), the time from which
its photo was current. It comes from the upload date in Slack's avatar URL
(`avatars.slack-edge.com/YYYY-MM-DD/…`, undocumented), from the start of the next UTC
day so a photo never shows early. The URL date is only trusted when it's before the
snapshot and after the latest earlier snapshot with a different `image_72`, otherwise
the snapshot time is used. It's resolved once at pack time, so slackback only compares.

## slackopy: channel view reads like Slack

Chronological, with consecutive messages grouped under one header
(`src/lib/groupMessages.ts`). Opens at the newest message without JavaScript:
links end in `#latest`, and `scroll-initial-target` covers bare URLs in Chromium.
This relies on images having their final size before loading (Slack's
`original_w`/`original_h`).

## Media: convert or probe at pack time, no transcoding

slackpack makes browser-friendly copies of images browsers can't show and records
video sizes, from the archived originals. Videos aren't transcoded, so HEVC only plays
where the browser supports it; with no client JS, players always link the file too.
