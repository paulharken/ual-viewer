# UAL Viewer

Offline, browser-based timeline visualiser for the Microsoft 365 Unified Audit Log.

Drop in a CSV export, get a virtualised, risk-classified timeline you can pivot by user or by file. Everything runs in the page — no server, no uploads, no external fonts or CDNs. Designed to live on an isolated analysis box.

![status](https://img.shields.io/badge/status-active-brightgreen) ![offline](https://img.shields.io/badge/runtime-browser--only-blue) ![deps](https://img.shields.io/badge/dependencies-none-lightgrey)

## Quick start

1. Download [`ual-timeline.html`](ual-timeline.html).
2. Open it in any modern browser (Chrome, Edge, Firefox, Safari).
3. Drag a UAL CSV onto the drop zone, or click to browse. Multiple files merge and dedupe automatically.

That's it. No install, no build step, no network calls.

## What it accepts

The parser handles the three CSV shapes you'll see in the wild:

- **Invictus IR Extractor Suite `Get-UAL`** — wrapper columns plus an `AuditData` JSON blob
- **`Get-UAL -AuditDataOnly`** — flattened, AuditData fields promoted to top-level columns
- **Purview portal export** — same `AuditData` blob shape as the Extractor Suite

Embedded commas, doubled quotes, and newlines inside the JSON blob are handled correctly. Rows without a usable timestamp are skipped and reported in the load status, so you'll know if anything didn't make it in.

## Workflows it's built for

**1. User activity walkthrough.** Pull a `userID` with the Invictus Extractor Suite, drop the CSV, pick the user from the dropdown on the **User activity** tab. The timeline scopes to everything that account did, in order, with risk-classified events.

**2. File / object focus.** Pull a SharePoint or OneDrive object with the Invictus Extractor Suite, drop the CSV, switch to the **File / object focus** tab, and search by filename or path fragment. Select a matching `ObjectId` to see a summary card (first seen, last seen, distinct users, distinct IPs, top operations) plus the scoped timeline.

**3. IP / session focus.** Switch to the **IP / session focus** tab and pick a source IP. The timeline scopes to everything that IP did *across all users* — the cross-account view that matters in BEC/ATO cases. IPs used by more than one account are sorted to the top and flagged (`⚠ N users`), and a configurable shared-session window marks rapid account switches from the same IP (`↔`) — the "one Tor exit, three mailboxes in ten minutes" pattern, surfaced at a glance.

## DFIR-relevant features

- **Risk classification.** Operations auto-bucket into categories with a high/med/low/info weighting. BEC inbox rules, illicit OAuth consent grants, `MailItemsAccessed`, send-as, anonymous sharing links, mass file download, role grants — all bubble up as high. Toggle "high-risk only" in the toolbar for a fast triage pass.
- **UTC native.** Timestamps are stored as UTC (UAL's native form). One-click toggle to local time for context; the date-range filter respects whichever you've selected.
- **Pivot from any record.** Click a user, IP, or object in the detail panel to refocus the whole view. Your current event stays selected so you can keep reading.
- **Virtualised timeline.** Only the visible window renders, so large exports don't choke the DOM.
- **Workload-aware.** Events are colour-tagged by workload (SharePoint, Exchange, AzureActiveDirectory, Teams, etc.) with per-workload filter chips.
- **Raw record on tap.** The detail panel shows the pretty-printed, syntax-highlighted `AuditData` JSON alongside the parsed summary.
- **Export view.** Dumps the currently filtered set to a flat CSV with the key fields, ready to drop into a report or timeline doc.

## Privacy

The page is fully self-contained. No webfonts, no analytics, no CDN-hosted libraries, no remote calls of any kind. Audit data never leaves the browser tab. Open the file with the DevTools network panel watching if you want to verify — there's nothing to see.

This makes it safe to run on isolated analysis VMs and on data subject to handling restrictions.

## Roadmap

Tracked as GitHub issues.

**Shipped:**
- ✅ [#1 RecordType-aware field mapping](https://github.com/paulharken/ual-viewer/issues/1) — per-RecordType IP / UA extraction for AAD/EntraID sign-in records, plus OAuth / token detail
- ✅ [#3 IP / session correlation view](https://github.com/paulharken/ual-viewer/issues/3) — cross-user pivot by source IP, multi-user-IP flagging, session-switch markers

**Open:**
- [#2 Web Worker async parsing](https://github.com/paulharken/ual-viewer/issues/2) — keep the UI responsive on multi-hundred-MB exports
- [#4 Configurable risk classification](https://github.com/paulharken/ual-viewer/issues/4) — analyst-editable rules, persisted locally
- [#5 Bookmarks and annotations](https://github.com/paulharken/ual-viewer/issues/5) — mark key events and export an annotated set for reporting
- [#6 Node / spider-web graph view](https://github.com/paulharken/ual-viewer/issues/6) — force-directed graph with a focal user/object/IP surrounded by everything connected to it
- [#7 Context window](https://github.com/paulharken/ual-viewer/issues/7) — ±N minutes around a selected event, across all users, for "pivot to the moment"

## Contributing

If you've got a de-identified header row from a real tenant pull — particularly for record types beyond Exchange and SharePoint — open an issue or PR. The field mapping in `normalise()` is the part most likely to drift as Microsoft adds new operations and record shapes.

## License

MIT.
