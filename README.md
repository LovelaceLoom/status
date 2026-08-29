# status

Public status feed for Lovelace Loom apps, served by GitHub Pages.

- Machine file: <https://lovelaceloom.github.io/status/status.json>
- Human page: <https://lovelaceloom.github.io/status/>

Two consumers:

1. **ToText iOS app** — fetches `status.json` on launch and when returning to the foreground.
2. **`index.html`** — renders the same file for humans (Hebrew and English).

## JSON contract

```json
{
  "schemaVersion": 1,
  "apps": {
    "totext": {
      "status": "ok",
      "detail": null,
      "updatedAt": "2026-08-29T00:00:00Z"
    }
  }
}
```

| Field       | Values                                                                 |
| ----------- | ---------------------------------------------------------------------- |
| `status`    | `ok` \| `outage` \| `maintenance`                                      |
| `detail`    | `null`, or a user-facing message: `{"he": "...", "en": "..."}`         |
| `updatedAt` | ISO-8601 UTC, e.g. `2026-08-29T14:30:00Z`                              |

**`updatedAt` doubles as the incident id.** The app shows the popup once per distinct `updatedAt` value — so **always bump `updatedAt` whenever you change `status` or `detail`**, or users who already dismissed an earlier popup will never see the new one.

### Optional fields (app v2.2.4+, still `schemaVersion: 1`)

All of these may be omitted — older app builds ignore them, and a malformed value degrades to "not shown" without hiding the incident itself.

| Field           | Values                                                                                       |
| --------------- | -------------------------------------------------------------------------------------------- |
| `phase`         | `investigating` \| `identified` \| `monitoring` \| `resolved` — small label on the incident card/banner; any other word shows no label |
| `nextUpdateBy`  | ISO-8601 UTC — rendered as "Next update by HH:MM" in the user's local time; a PAST time still renders, so keep the promise or bump it |
| `postmortemURL` | `https://` link only — offered as "What happened" on the resolved banner after the incident clears |
| `announcement`  | `{"id": "...", "title": {"he": "...", "en": "..."}, "body": {"he": "...", "en": "..."}, "url": "https://..."?}` — see below |

Example incident entry with everything:

```json
{
  "schemaVersion": 1,
  "apps": {
    "totext": {
      "status": "outage",
      "detail": {"he": "...", "en": "..."},
      "updatedAt": "2026-08-29T14:30:00Z",
      "phase": "identified",
      "nextUpdateBy": "2026-08-29T16:00:00Z",
      "postmortemURL": "https://lovelaceloom.github.io/status/postmortems/2026-08-29.html"
    }
  }
}
```

### Resolved banner (automatic — nothing to publish)

A device that showed an incident remembers its `updatedAt` (and `postmortemURL`, if any). When the file later returns to `"ok"` — or the `totext` entry disappears — that device shows one calm "Service is back to normal." banner, with a "What happened" link when the incident carried `postmortemURL`. Dismissed once, it never returns for that `updatedAt`. So: publish the postmortem link **on the incident itself** (it is fine to add it on the final update, together with `phase: "resolved"`), then flip to `ok`.

### Announcements (neutral news — not an incident)

`announcement` inside the `totext` entry is for proactive communication: news, tips, new-version notes. It renders as a calm, non-alarming card — never the warning style — and a declared incident always outranks it (the card waits until the incident clears; it is not lost).

- **`id` is the announcement's identity**: shown once per distinct `id`, dismissible; publish a NEW `id` to raise a new card (re-publishing the same `id` re-shows nothing for users who dismissed it).
- `title` is required (he/en, en is the fallback); `body` likewise; `url` is optional and must be `https://`.
- An announcement may ride any `status`, including `"ok"` — that is its normal home.

```json
"announcement": {
  "id": "2026-09-widget-tip",
  "title": {"he": "...", "en": "New: lock-screen widget"},
  "body": {"he": "...", "en": "Add ToText to your lock screen to start recording in one tap."},
  "url": "https://totext.app/news/widget"
}
```

## Signed control (emergency API redirect — app v2.2.5+)

The file has **two trust domains**:

- **Unsigned = communication.** Everything documented above — `status`, `detail`, `phase`, `announcement`, … — is editable by anyone with write access, on purpose: incident comms must be fast, and the worst a forged edit can do is show users a message.
- **Signed = traffic control.** Anything that steers where the app sends data lives in a top-level `"control"` block, detached-signed with the owner's offline Ed25519 key. An unsigned `apiBaseURL` anywhere in this file does **nothing** — the app only ever reads it out of a verified `control` payload, because whoever could redirect the API would silently receive users' audio.

```json
"control": {
  "payload": "<base64 of canonical JSON>",
  "signature": "<base64 Ed25519 detached signature>",
  "keyId": "k1"
}
```

The decoded payload is `{"apiBaseURL": "https://…"?, "signedAt": <unix seconds>}`. The app verifies the signature against public keys pinned in the binary (`keyId` picks the slot, `k1`/`k2`), then applies `apiBaseURL` instead of its built-in API origin. `signedAt` is an anti-rollback floor — devices remember the highest value they accepted and refuse anything older — so a captured redirect block can't be replayed after the all-clear. Verification failures of any kind are silent: the app just keeps using its built-in URL. The `status.json` fetch itself never follows the override.

**Never edit `control` by hand.** Produce it in the app repo with the owner key:

```sh
bash scripts/status-sign.sh https://backup-api.example.com   # emergency redirect
bash scripts/status-sign.sh --clear                          # signed return to built-in
```

Each run signs a fresh `signedAt`; paste the printed block into this file as a sibling of `"apps"` and commit. Prefer publishing a `--clear` block over deleting `control` when the incident ends — the signed all-clear advances the floor, so the old redirect can never be replayed. (One-time key ceremony: `bash scripts/status-keygen.sh` in the app repo; the private key lives only on the owner's machine and in the password manager.)

## Incident ritual

### Declare an outage

No clone needed — edit in the GitHub web UI:
<https://github.com/LovelaceLoom/status/edit/main/status.json>
Set `status` to `"outage"`, fill `detail` (he + en), bump `updatedAt`, commit as `status: outage`.

Or from a terminal:

```sh
git clone https://github.com/LovelaceLoom/status.git
cd status
date -u +%Y-%m-%dT%H:%M:%SZ    # copy this into updatedAt
# edit status.json:
#   "status": "outage",
#   "detail": {"he": "אנחנו מטפלים בתקלה", "en": "We are working on an issue"},
#   "updatedAt": "<the timestamp above>"
git commit -am "status: outage"
git push
```

(For planned work use `"maintenance"` instead of `"outage"`, commit as `status: maintenance`.)

### All clear

Web UI (same link), or:

```sh
cd status
git pull
date -u +%Y-%m-%dT%H:%M:%SZ    # copy this into updatedAt
# edit status.json:
#   "status": "ok",
#   "detail": null,
#   "updatedAt": "<the timestamp above>"
git commit -am "status: ok"
git push
```

## Notes

- GitHub Pages sits behind a CDN that caches for up to ~10 minutes; expect a short delay before clients see a change.
- Keep `schemaVersion` at `1`; bump it only alongside an app release that understands the new shape.
