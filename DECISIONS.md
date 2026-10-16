# Decisions

What to assume before starting new work: current decisions only, not history. Edit
or delete entries when they change; point at code instead of restating it. None of
this is final: question an entry when new work gives a reason to.

## Rendering & auth

slackopy is SSR, checking the session per page. Fully static behind an Astro
middleware auth-gate is the goal, done in one pass so the site is never ungated.

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
