# Decisions

What to assume before starting new work: current decisions only, not history. Edit
or delete entries when they change; point at code instead of restating it. None of
this is final: question an entry when new work gives a reason to.

## Rendering & auth

slackopy is a static site, rebuilt at startup and daily after slackpack's sync. An
auth proxy in front gates every path, archived files included, redirecting straight
to FusionAuth login: no public pages, no login screen. Logins last 30 days but are
rechecked against FusionAuth hourly.

## slackpack stores Slack's raw JSON

A minimal own client (`internal/slackapi`) rather than a typed library, so no field
Slack sends is lost. Only volatile keys (statuses, huddles, dormancy) and the token owner's `saved`
state are stripped, so snapshots don't churn.

## Archive in slackpack, embed pointers in JSON

Files and avatars go into unversioned tables (Slack IDs/URLs are immutable), with
pointers embedded in message/user JSON at pack time, so slackback never joins them.
Anything gone from Slack (403/404/410) is never retried.

## Migrations are forward-only

No down-migrations; rollback is a Postgres backup/restore, since an untested
down-script can't be trusted against irreplaceable data.

## Production setup is a generated quickstart

`quickstart.sh` generates the deployment directory and preserves secrets on re-run.
No bundled proxy or published ports; it joins the admin's proxy network.

## Snapshots: latest for channels, as of message time for users

Channel views show the latest snapshot; archived/deleted channels stay listed.
Authors, mentions and reactors use the user snapshot valid at the message ts (daily,
so up to a day late). Avatars switch at the photo's upload date from its URL
(undocumented), so they can predate names; placeholder before that.

## slackopy: channel view reads like Slack

Grouped and chronological, opening at `#latest` without JS. That relies on all media
having known dimensions before loading.

## Media: convert or probe at pack time, no transcoding

slackpack makes browser-friendly copies of images browsers can't show and records
video sizes, from the archived originals. Videos aren't transcoded, so HEVC only plays
where the browser supports it; with no client JS, players always link the file too.

## Media: Slack thumbnails for load time

Images show Slack's `thumb_960` and videos its `thumb_video` as a poster (with
`preload="none"`); images lazy-load. Being Slack's, they only backfill files still in
Slack, else the full file shows.
