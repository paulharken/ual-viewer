# Entity Graph View — Maltego-style relationship visualisation (issue #6)

## Context

`ual-viewer` is a single-file, offline, vanilla-JS visualiser for Microsoft 365
Unified Audit Logs (`ual-timeline.html`). Today every view is **linear or
single-entity-pivoted**: the analyst picks one user / object / IP and the
timeline filters to that scope (`F.tab` + `F.user`/`F.obj`/`F.ipsel`). What it
*can't* show is **what is connected to what** — the thing an analyst reaches for
in Maltego: a focal entity in the centre surrounded by everything it touches.

GitHub **issue #6 ("v2: Node / spider-web graph visualisation")** specifies
exactly this: a force-directed graph with a single focal node (user, object, or
IP) and weighted, risk-coloured edges to the entities it connects to. "This file
had 47 events, but they're from 3 distinct users across 12 IPs" is one shape in
a graph and a scroll-through in a timeline.

**Decisions (confirmed with user):** deliver the **full v2** (all three centre
modes + all interactions) in one iteration, surfaced as a **new "Graph" tab**
alongside User / File / IP. Hard constraint from the issue: **stay single-file,
zero dependencies** — a small custom `<canvas>` force-directed layout, no
Cytoscape/Sigma/d3.

## Approach

Add a 4th tab, `graph`, that reuses the **current focal entity** as its centre
node and renders a custom canvas force-directed graph. Re-centring on a satellite
updates the shared pivot state (`F.user`/`F.obj`/`F.ipsel`) so the Graph tab and
the timeline tabs stay in lockstep — drilling in the graph and then switching
back to the timeline lands on the same entity.

### 1. UI / DOM additions

- **Tab button** in `.tabs` (line ~263): `<button data-tab="graph">Graph</button>`.
- **Pivot block** `<div class="pivot" id="pivot-graph">` (after `#pivot-ip`,
  line ~304): a centre-node selector row — entity-type segmented control
  (user / object / IP) + a value dropdown/search, a "top-N satellites" control,
  a "reset view" button, and a small legend (entity-type colours, red = high-risk
  edge). It reflects/sets `F.user`/`F.obj`/`F.ipsel`.
- **Canvas surface** inside `.body`: a `<div class="graph-wrap"><canvas id="gcanvas">`
  shown only on the graph tab (toggle a class on `.body`, mirroring how
  `detail-collapsed`/`raw-collapsed` hide panels). The timeline-wrap is hidden
  while the graph tab is active; the detail + raw panels stay visible so clicking
  a node/edge can still drive the inspector.
- **CSS**: reuse existing custom properties (`--bg`, `--txt`, `RISK_COLOR`,
  `wlColor`) for canvas fills; add `.graph-wrap`, `#gcanvas`, legend/control styles
  in the embedded `<style>`.

### 2. Data layer — build the graph model from `EVENTS`

New `buildGraph(centerType, centerValue)` → `{nodes, edges}`:

- **Centre node**: `{id, type:centerType, label, kind:'center'}`.
- **Satellite nodes**, by centre mode (mirrors issue #6 + reuses existing
  aggregation patterns):
  - **user-centred** — satellites: distinct `ip`, `obj`, `op`, `wl`.
  - **object-centred** — satellites: distinct `user`, `ip`, `op`, parent `site`.
  - **IP-centred** — satellites: distinct `user`, `obj`, `op`.
- Iterate the events for the centre (same filter shape as `renderObjSummary`
  line ~1066 / `renderIpSummary` line ~1110: `EVENTS.filter(e=>e.<field>===value)`),
  aggregating into a `Map(satelliteKey → {type, count, maxRisk})`:
  - **edge weight** = event count between centre and satellite.
  - **edge risk** = highest-risk op contributing (`high` > `med` > `low` > `info`),
    using the existing `risk` field; red edge if any contributing event is `high`.
  - keep the **member event references** per edge so an edge-click can scope the
    timeline (or recompute on click via the same filter — cheaper to store).
- **Cap to top-N** satellites per group by `count` (default ~8–12 per group,
  overall budget ~200 nodes per issue), collapsing the long tail into a single
  expandable **"+N more"** pseudo-node; clicking it raises the cap for that group.
- Respect the **active filters**? Build from `EVENTS` for the full relationship
  picture, but offer a toggle to honour `F.risks`/`F.wls`/date/search so the graph
  can follow the filtered scope. (Default: full EVENTS; document the toggle.)

### 3. Force-directed layout (custom, no deps)

- Node physics state: `{x, y, vx, vy, pinned}`. Centre node pinned at canvas
  centre. Satellites seeded on a ring around it.
- Per-tick: O(n²) **repulsion** between all nodes (Coulomb-style, fine at ≤200
  nodes), **spring attraction** along edges (rest length scaled by inverse weight
  so heavier edges pull closer), mild **centering** force, **velocity damping**.
- `requestAnimationFrame` loop; **stop when settled** (total kinetic energy below
  a threshold) to save CPU; restart on interaction (drag, recentre, expand,
  resize). DPR-aware canvas sizing (`devicePixelRatio`, `ResizeObserver` on the
  wrap).

### 4. Rendering (canvas 2D)

- **Edges** first: width ∝ weight (clamped), colour by edge risk (red high /
  amber med / blue low / grey info via `RISK_COLOR`), brightened on hover.
- **Nodes**: filled circle coloured by entity type (user / ip / obj / op / wl —
  workload uses `wlColor`), centre node larger/outlined; label drawn beside node
  (truncate with `shortObj` for object ids).
- **Hover/selection** states drawn each frame; show an edge's event count in a
  small tooltip/HUD on hover.

### 5. Interactions

- **Hit-testing** in canvas coords for mousemove/click/contextmenu (nearest node
  within radius; nearest edge segment for edge clicks).
- **Hover satellite** → highlight connecting edge + show event count.
- **Click satellite** → **re-centre**: set the matching `F.*` focus and rebuild
  the graph; keeps Graph tab and timeline pivots in sync. (User→ user dropdown;
  obj→ `selectObject`-style; ip→ `selectIp`-style — but without leaving the graph
  tab.)
- **Click edge** → scope the timeline to just those events: switch to the
  centre's natural timeline tab with the centre pivot set, or set a transient
  edge-scope filter, then `applyFilters()` + jump to the timeline.
- **Right-click / long-press node** → toggle `pinned` (fixed during simulation).
- **Drag node** → reposition (auto-pin while dragging).
- **"+N more"** → expand the long tail for that satellite group.

### 6. Wiring & integration

- Extend `setTab()` (line ~1168) to toggle `#pivot-graph` and the
  graph/timeline body classes, and to **lazily build + start** the graph when
  entering the tab, and **stop the RAF loop** when leaving (CPU hygiene).
- Add the `graph` case so the existing tab click wiring (line ~1203) just works.
- Add `'graph'` handling so the centre defaults sensibly from whatever focal
  entity is already set (e.g. coming from the User tab with `F.user` set →
  user-centred on that user).
- `applyFilters()` (line ~699) needs no pivot change for graph (graph is its own
  scope), but if the graph honours filters, call a `renderGraph()`/`rebuild` hook
  at its end when `F.tab==='graph'`.
- `btn-reset` (line ~1291): stop the loop and clear graph state alongside the
  existing resets.
- Persist lightweight prefs (centre entity-type, top-N) under `ual-viewer-graph-*`
  localStorage keys, matching existing namespacing.

### Functions to ADD
- `buildGraph(type, value)` — graph model from EVENTS (reuses summary filter
  patterns; `IPUSERS`, `OBJINDEX` where helpful).
- `graphLayoutTick()` / `startGraphSim()` / `stopGraphSim()` — physics + RAF.
- `renderGraph()` / `drawGraph(ctx)` — canvas draw.
- `graphHitTest(x,y)`, plus mouse/contextmenu/drag handlers.
- `recenterGraph(node)` and `scopeTimelineToEdge(edge)`.

### Functions to MODIFY
- `setTab()` (~1168), tab wiring (~1203), `applyFilters()` tail (~718–724),
  `btn-reset` (~1291), plus the HTML/CSS additions noted above.

## Verification

1. Open `ual-timeline.html` in a browser; drop a sample UAL CSV (an export with
   several users sharing IPs / touching shared objects best exercises edges).
2. **User-centred**: User tab → pick a user → Graph tab. Confirm centre = user,
   satellites = its IPs/objects/ops/workloads, edge widths track event counts,
   red edges where a high-risk op exists.
3. **Re-centre**: click an object satellite → graph re-centres on the object,
   satellites become the users/IPs that touched it. Switch to the File tab and
   confirm the object pivot followed.
4. **Edge click** → lands on the timeline scoped to those events.
5. **Object- and IP-centred** modes via the centre selector; **pin** a node
   (right-click), **drag** nodes, **expand** "+N more".
6. **Performance**: load a large export, confirm the sim settles and stays
   interactive at ~200 visible nodes and stops spinning the CPU once settled and
   when leaving the tab.
7. Confirm still fully offline / single-file (no network requests in devtools).

## Build order (incremental, single PR)

1. **Scaffold** — Graph tab + `#pivot-graph` + canvas in `.body`; `setTab()` shows
   it; empty canvas renders.
2. **Data + draw** — `buildGraph()` for the user-centred mode; static draw of
   nodes/edges (no physics yet) to validate the model and colours/weights.
3. **Physics** — force loop + RAF, settle-and-stop, DPR/resize handling.
4. **Interactions** — hover tooltip, hit-testing, click-to-recentre, edge→timeline,
   right-click pin, drag, "+N more" expand.
5. **All centre modes** — object-centred and IP-centred; centre-type selector;
   sensible default centre from the incoming focal entity.
6. **Polish** — legend, localStorage prefs, `btn-reset` integration, perf pass.

Each step keeps the file runnable so it can be verified as it lands.
