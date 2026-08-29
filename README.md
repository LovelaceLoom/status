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
