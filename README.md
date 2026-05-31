# UAL Viewer

Offline, browser-based timeline visualiser for the Microsoft 365 Unified Audit Log.

Drop in a CSV export, get a virtualised, risk-classified timeline you can pivot by user or by file. Everything runs in the page — no server, no uploads, no external fonts or CDNs. Designed to live on an isolated analysis box.

![status](https://img.shields.io/badge/status-active-brightgreen) ![offline](https://img.shields.io/badge/runtime-browser--only-blue) ![deps](https://img.shields.io/badge/dependencies-none-lightgrey)

## Quick start

Two ways to run it — both run entirely in your browser; **your audit data is never uploaded either way.**

- **Hosted:** open **https://paulharken.github.io/ual-viewer/** — nothing to install.
- **Local / offline:** download [`ual-timeline.html`](ual-timeline.html) and open the single file in any modern browser. This is the one to use on an **isolated or air-gapped analysis box** — it needs no network at all.

Then drag a UAL CSV onto the drop zone, or click to browse. Multiple files merge and dedupe automatically.

> The hosted version is a convenience mirror: loading the *page* is a request to GitHub, but the CSVs you load are parsed locally and stay in the tab. For the most sensitive work, prefer the downloaded local file so even opening the tool leaves no trace.

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

**4. Time slice.** The **Time slice** tab is a scrubbing timeline: a horizontal time axis with a centre **playhead** marking the exact point in time you're browsing, and events fanned out by timestamp into colour-coded **per-workload lanes** stacked up the screen, so you can see everything that happened in the seconds either side of a chosen moment. Pick a **window** of 5 / 10 / 20 seconds, then **drag the axis** to scrub left and right through time. A full-log **overview bar** on top shows where the current slice sits in the whole log — drag its window box to jump, or click anywhere on it to seek. Click a mark to inspect the full record. Respects all the active filters (risk, workload, search, date range) and the UTC/Local toggle. Custom canvas, no dependencies.

## DFIR-relevant features

- **Risk classification.** Operations auto-bucket into categories with a high/med/low/info weighting. BEC inbox rules, illicit OAuth consent grants, `MailItemsAccessed`, send-as, anonymous sharing links, mass file download, role grants — all bubble up as high. Toggle "high-risk only" in the toolbar for a fast triage pass.
- **UTC native.** Timestamps are stored as UTC (UAL's native form). One-click toggle to local time for context; the date-range filter respects whichever you've selected.
- **Pivot from any record.** Click a user, IP, or object in the detail panel to refocus the whole view. Your current event stays selected so you can keep reading.
- **Virtualised timeline.** Only the visible window renders, so large exports don't choke the DOM.
- **Workload-aware.** Events are colour-tagged by workload (SharePoint, Exchange, AzureActiveDirectory, Teams, etc.) with per-workload filter chips.
- **Workload-specific context.** The inspector decodes the `AuditData` per workload and surfaces the fields that matter — leading with the security signals — instead of leaving them in the JSON. Covered workloads:
  - **MicrosoftTeams** — meeting join/leave + duration, device, attendee lists with roles, **external/guest flagging** (UPN `#EXT#`/org ≠ tenant, with the external domain shown), Federated and Private/Shared channel call-outs, links shared, app installs. (Grounded in [HubTou/tala](https://github.com/HubTou/tala).)
  - **Exchange** — delegate/admin mailbox access (LogonType + actor ≠ owner), `MailItemsAccessed` Bind/Sync + throttle "log gap" warning + accessed subjects, send-as, and inbox-rule forward/redirect/delete flags.
  - **SharePoint / OneDrive** — file + path, **anonymous-link** and **external-share** flags (Guest / domain mismatch), permission, sensitivity label, sync device.
  - **AzureActiveDirectory (Entra)** — directory-change `ModifiedProperties` diffs, flags for role grant / app consent (high-risk scope) / SPN credential add / MFA disable / domain-federation. Sign-ins fall through to the OAuth/Token section.
  - **Yammer / Viva Engage** — data export, private-content-mode toggle, admin-role changes, retention flag.
  - **Endpoint DLP** — egress activity, device, sensitive-info types, SHA256, USB make/model/**serial**, enforcement mode.
  - **Sensitivity labels (MIP)** — `SensitivityLabelAction` / `SensitivityLabeledFileAction` records (these carry `Workload=PublicEndpoint`): content type + app, label apply/change (old → new), apply source, document encryption + owner, recipients with external flagging. Flags **label changed**, **encryption removed**, **external recipients**.
  - **Compliance DLP / auto-labeling** — `MipLabel` / `DlpRuleMatch` records: policy + rule + severity, matched conditions (incl. `AccessScope: IncludeExternalUsers`), labels, sender/recipients, external-viewable, attachments. Flags **DLP policy match**, **external recipients**, **high severity**.

  Schemas sourced from the M365 Management Activity API / Purview audit docs + DFIR references; extraction is tolerant (only shows present fields).
- **Two inspector panels, your choice.** The decoded **detail** panel and the pretty-printed, syntax-highlighted **raw AuditData** panel are separate and sit side by side. Toggle each independently from the top bar (`hide detail` / `hide raw`) — show one, both, or neither; hiding both gives the timeline the full window width. Drag either panel's left edge to resize it (double-click the edge to reset) when long object IDs or UPNs need more room. Timeline columns are drag-resizable too. All choices persist.
- **Keyboard-driven.** `↑`/`↓` (or `j`/`k`) walk the timeline, `PgUp`/`PgDn` jump a viewport, `Home`/`End` go to the ends — the selected row scrolls into view and the panels update live.
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
- [#8 Workload-specific detail sections](https://github.com/paulharken/ual-viewer/issues/8) — Teams shipped; Exchange / SharePoint / AAD decoding next

## Contributing

If you've got a de-identified header row from a real tenant pull — particularly for record types beyond Exchange and SharePoint — open an issue or PR. The field mapping in `normalise()` is the part most likely to drift as Microsoft adds new operations and record shapes.

## License

MIT.
