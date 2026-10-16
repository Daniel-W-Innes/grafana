# Dashboard style guide

**Status: v1.1 (2026-09-07).** All forks settled same day (U1, D1, P1/P2, T1, T2 — see
Decisions log); v1.1 folds in external inspiration: purpose/context (§0), threshold
discipline (§1), grid & graph rules (§4), annotations & maintenance (§9).
The guide is normative from that date for new panels and any panel touched in a patch
pass. Changes to the guide itself land via the Decisions log at the bottom.

This guide governs **new panels and dashboards, and any panel we touch** during a patch
pass. Existing untouched panels (imported community dashboards especially) are migrated
row by row, not wholesale. Source of truth is the live Grafana; changes land as MCP
`update_dashboard` patch ops, never as blind file edits (see CLAUDE.md).

---

## 0. Purpose & context

Folded in 2026-09-07 from Grafana Labs' dashboard-design guidance ("know your audience",
"tell a story") and Wikimedia's SRE dashboard conventions (legend panels). Sources at the
bottom of this file.

- **Dashboard `description`** — one sentence: what the dashboard watches and what
  "normal" looks like, i.e. the decision it supports when something is off. This is the
  first thing read when an alert points at a dashboard; it answers "do I need to act?"
  faster than any panel. Written once when the dashboard is created or touched.
- **`tags`** — lowercase tags naming the monitored services (`backup`, `arr`,
  `network`, `probe`, `media`, `security`). This is how dashboards get found later; the
  fleet today is only findable by filename.
- **Provenance legend, only where non-obvious** — if a dashboard reads metrics whose
  source isn't obvious (security: konnected sensors via InfluxDB; borgmatic: textfile
  collector under `job="node_exporter"`; iperf3: probe exporter), a one-panel legend or
  a `links` entry naming the source earns its place. Skip it where the source is
  obvious (`job="prometheus"`, node_exporter); a legend that restates the query is
  noise. Never put provenance in row titles — rows are for data.

## 1. Palette & thresholds

**Color policy (settled 2026-09-07 — P1/P2).** Color does not need to be uniform
fleet-wide. Two tiers:

1. **Panels whose colors carry health/status meaning — where a threshold is obviously
   "good/bad/warn" — use the house scale** below. This is most stats/gauges.
2. **Panels without obvious thresholds** (weather values, categorical breakdowns,
   distributions) **may use any sensible current color** — no need to force the house
   palette. openweathermap's weather palette stays as it is.

Banned everywhere: **legacy colors** — `rgba(…)` literals and the old `dark-*` /
`semi-dark-*` palette variants (theme-fragile leftovers from imports; found in
node_exporter_full, exportarr, uptime). Migrate them on touch: to house colors where
the panel is thresholded, to a plain named color otherwise. Hex equivalents of house
colors (smokeping's `#16a34a`/`#dc2626` etc.) are acceptable and not "legacy"; when such
a panel is touched for other reasons, prefer the named color for theme portability.

| Color | Meaning | Use |
|---|---|---|
| `green` | OK / healthy | base step, success states |
| `orange` | warning tier | middle step of a 3-step scale |
| `red` | bad / critical | top step, failure states |
| `#EAB839` | amber accent | single-color "watch this" gauges (borrowed from Grafana's classic amber) |
| `blue` | cold / informational | temperatures and weather only; not a health state |
| `transparent` | no fill | spare/background styling only, not for thresholds |

Threshold rules (house-scale panels):

- Threshold scales run base → `green`, then `orange`, then `red` — never skip tiers in
  a health scale, and never use a color twice in one scale.
- Every threshold step list starts with a base step (`color: "green"`, value `-Infinity`
  or the scale floor for gauges — see §2) so values below the first named threshold stay green.
- Gauges set explicit `min`/`max` on `fieldConfig.defaults` so the arc matches the scale,
  and keep `showThresholdMarkers: true`.
- Do not attach colors to series in `fieldConfig.overrides` when a threshold scale would
  express the same thing — health via thresholds, categorical data via overrides.

### Threshold discipline (settled 2026-09-07)

- Thresholds come from the thing being measured, not from round numbers: the backup
  schedule (borgmatic's check ages), a real capacity limit, or percentiles of observed
  history. Tiers invented without semantics will lie in one direction or the other.
- Calibrate tiers against real data: run the query over a representative range before
  finalizing and check the tiers split the actual distribution sensibly. **If
  everything is red, nothing is red** — a tier set that fires in the steady state is
  wrong.
- Keep the normal state calm. Most panels green most of the time; a panel that sits
  amber/red at rest is mis-thresholded or monitoring the wrong thing.
- The same metric keeps the same color across panels and dashboards: where a query
  returns several series and the colors matter (in/out, up/down, per-zone), pin series
  colors via `fieldConfig.overrides` instead of letting the palette re-roll per load.

## 2. Units

Fleet today spells bytes-per-second four different ways (`Bps`, `binBps`, `decbytes`,
plus `bps`/`decbits`) and mixes base-2 and base-10. One spelling per quantity.

Use Grafana unit IDs exactly; never free-text units (emoji included — settled 2026-09-07).

| Quantity | Canonical unit ID | Notes |
|---|---|---|
| Storage / memory / filesystem | `bytes` (base 2) | `decbytes` is legacy → migrate |
| Transfer rate, bytes/s | `binBps` | `Bps` and `decbytes`-as-rate are legacy → migrate |
| Transfer rate, bits/s | `binbps` | `bps`, `decbits` are legacy → migrate |
| Ratio 0–1 (metric already a ratio) | `percentunit` | e.g. `…_utilization_ratio` |
| Ratio scaled ×100 | `percent` | expression ends in `* 100` |
| Duration | `s`, `ms`, `d` | `d` only for uptime-style "days since/until" |
| Time of last event | `dateTimeAsIso` (absolute, on gauges) / `dateTimeFromNow` (relative, on stats) | don't switch per panel idly |
| Temperature | `celsius` | |
| Clock / frequency | `hertz` | |
| Power | `watt` | |
| Pressure (weather) | `pressurehPa` — divide Pa metrics by 100 in the query | raw Pa (`pressurepa`-style, ~101325) is unreadable |
| Rain depth (weather) | `lengthmm` — multiply m metrics by 1000 | 0.0004 m is unreadable; 0.4 mm is not |
| Wind speed | `velocityms` | |
| Direction | `degree` | |
| Plain counts (goroutines, series, players) | `none` | thousands separators via `short` if large |
| Percentages that aren't ratios (SLA) | `percent` | |

Rules:

- If the metric ends in `_ratio` the unit is `percentunit`; if the expression scales by
  `* 100` the unit is `percent`. Never mix these on the same dashboard's panels.
- `decimals`: `1` for `bytes`/`binBps`, `1–2` for `percentunit`, `0` for big `short`
  counts, sensible per panel otherwise. Set explicitly rather than inheriting the
  default where display matters.
- No unit in the panel **title** when the unit is set in `fieldConfig` (see §3 — a
  "GPU Utilization %" stat with `percentunit` shows "%" twice).

**Settled 2026-09-07 — no emoji.** Emoji are banned everywhere: units, titles,
descriptions. Navidrome's four emoji-unit stats (`Users`/`Songs`/`Albums`/`Artists`,
panels 40/42/43/44) drop their unit to `none` when the dashboard is next touched.

## 3. Titles & naming

Fleet today mixes strict TitleCase (iperf3, mc_monitor, exportarr) with sentence case
(borgmatic, prometheus), sometimes inside the same dashboard (navidrome). Panel keys are
uniformly `panel-<n>` = numeric `spec.id` — keep that; it is not up for debate.

**Settled 2026-09-07 — T1: Title Case.** Every significant word is capitalized,
including words after a hyphen (`Probe Status Over Time`, `Reverse Retransmits`,
`Out-of-Order Samples`). Only the short function words `a`, `an`, `the`, `and`, `or`,
`by`, `for`, `in`, `of`, `on`, `to`, `with` stay lowercase when they fall mid-title
(`Queries by Indexer`, `Retention and Space`); they capitalize at the start of a title.
Acronyms (`CPU`, `GPU`, `RSS`, `FD`, `FDs`, `WAL`, `TSDB`, `HTTP`, `SLA`, `RTT`) and
proper nouns are always capitalized. borgmatic and prometheus currently use sentence
case (`Samples ingested`, `Archive days`) and migrate to Title Case as their panels are
touched. Row titles are prose headings and stay sentence case where they read as prose
(`Checks — time since each check type last passed`); row titles that are target labels
use the target's own casing.

Title rules (settled):

- Capitalize acronyms and proper nouns, per above.
- No unit or `%` in the title when the unit is set in fieldConfig (`GPU Utilization %` →
  `GPU utilization`). `RTT Mean`/`Fan Speed %` type titles count too.
- No trailing/leading whitespace on titles that exist (found: `Http Code ` in traefik).
- Titles are **recommended, not required** (settled T2) — untitled panels are a
  legitimate pattern for chrome/status panels (exportarr's 29, plus stragglers in
  smartctl/security/smokeping). Where a panel *does* carry a title, every rule in this
  section applies to it.
- No `$variable` interpolation in panel titles (traefik's `2xx over $interval`). Put the
  context in the query and the meaning in a static title; row titles *may* interpolate
  (see §4).
- No emoji, unicode symbols, or decorative punctuation in titles.

### Current/history pairs (the duplicate-title problem)

Five dashboards today show the same metric twice — a current stat/gauge and its history
timeseries — both titled identically (`Temperature` ×2 in nvidia, `Songs` ×2 in
navidrome, `Latency` ×2 in uptime). mc_monitor/iperf3 already solved this. Preferred
pattern when both a current and a history panel exist:

- Current panel: `X` (stat or gauge, thresholds/min-max set).
- History panel directly beneath it, same column width: `X Over Time` (timeseries).
- The pair is the unit of layout: same `x`/`width`, consecutive `y` — see §4.

## 4. Layout & display

- Prefer `RowsLayout` with **named rows** for anything over ~8 panels. Every row gets a
  title (found: blank row titles in navidrome and uptime).
- First row carries the status summary (the stat/gauge strip), mirroring borgmatic and
  prometheus ("Latest backup", "Process").
- Multi-target dashboards put the target in the **row title**, with the row collapsed —
  exportarr's `Prowlarr - ${prowlarr_instance}`, `SABnzbd - ${sabnzbd_instance}`
  collapsed rows are the fleet's best pattern for this. Row titles may interpolate
  variables; panel titles may not (§3).
- Detail that is rarely watched lives in **collapsed** rows (`[collapsed]` state), as
  node_exporter_full does with 14 of its 16 rows.
- Flat `GridLayout` is for small single-purpose dashboards (≤ ~8 panels). iperf3 (18)
  and nvidia (23) are flat today — acceptable, but if we touch them anyway, giving them
  rows is an option.
- Panels in a row share height; current/history pairs share a column (§3).
- Importance flows top-left ("Z" scan): the status strip is the first row, and within a
  row the headline panel sits left of supporting ones. Everything matters, not equally.
- Keep the grid rhythm: panel heights in multiples of grid units (timeseries 8–12);
  a key timeseries may span the full 24 columns, comparisons and pairs run half (12)
  or narrower; stat/gauge strips share widths so columns align across rows.
- Structural invariants (never regress): every element key is `panel-<n>` with matching
  numeric `spec.id`; every `ElementReference` resolves; every element is placed exactly
  once; no overlapping grid items; no `range: true` + `instant: true` together (Grafana
  "Both" mode doubles every series).

### Timeseries display rules (settled 2026-09-07)

- Y-axis is zero-based unless there is a reason to zoom a small-delta metric
  (RTT/jitter graphs may start above zero).
- No area fill unless the graph is stacked; stack only when the total is the signal.
- Bounded graphs (percent / percentunit that cannot exceed 100 %) clamp Y max at 100.
- No more than ~5 series per graph — one graph answers one question. Beyond that, split
  the panel or show a stacked summary instead of a rainbow.
- No-data renders as a gap, not a zero — unless zero is the honest value. (uptime's
  SLA expression deliberately renders down-time as 0; that is semantic, and it is why
  the expression exists rather than a null-connecting setting.)

## 5. Queries & datasources

- **Settled 2026-09-07 — D1: every query references the dashboard's datasource
  variable**, `datasource: {"name": "${ds_prometheus}"}` (the reference lives in the
  `name` field). One variable per datasource type, named `ds_<type>`: `ds_prometheus`,
  `ds_influxdb` (security.json has both). The variable definition itself is the only
  place a concrete datasource is named (current Prometheus UID `PBFA97CFB590B2093`);
  copy the variable shape from an existing dashboard (mc_monitor/exportarr/traefik)
  when building a new one. Six dashboards still pin the UID per-query (borgmatic,
  iperf3, navidrome, nvidia, openweathermap, prometheus) and migrate as they're
  touched; uptime.json gains the variable (its queries currently name no datasource).
- `range: true` only, never `instant`, never both. House style has no `instant` key at all.
- Selectors: pin `job` where the job name is stable (`job="node_exporter"`,
  `job="prometheus"`), target the instance via a dashboard variable
  (`instance="$node"`, `uuid="$gpu"`, `monitor_name=~"$monitor"`) — or, where there is
  exactly one target that must never silently widen, pin it explicitly
  (`instance="onion.lc.brotherwolf.ca:9100"`, as borgmatic does for its check metrics).
  Prefer `=~"$var"` over `="$var"` when the variable offers `All`.
- `legendFormat`: `__auto` when a query returns one series; an explicit format
  (`{{label}}`) when it returns several.
- No dashboard depends on the *default* datasource silently — every query names one
  (found: uptime.json queries carry no datasource at all).

## 6. Variables

- Only add a variable that genuinely parameterizes the dashboard (`job`, `node`, `gpu`,
  `monitor`, app instance). Don't add one for a single fixed target.
- Name variables by what they select: `node` (host), `job`, `gpu`, `monitor`,
  `target`.
- Datasource variables (`ds_prometheus`, `ds_influxdb`, …) are required on every
  dashboard per D1 (§5), even single-datasource ones — they keep dashboards portable
  when a datasource is replaced. Set `hide: "hideVariable"` on the spec if you never
  switch them (existing dashboards default to `"dontHide"`), but keep them defined.
  `hide` is an enum (`dontHide` | `hideLabel` | `hideVariable`) — `"variable"` is not
  a valid value and git-sync rejects dashboards carrying it (see 2026-10-16 decision).
- Query variables get `includeAll` + a sensible `allValue` only when the panels' regex
  matchers (`=~"$var"`) can use it.

## 7. Time & refresh

Fleet today spans 30m (nvidia) to 30d (navidrome) with no dashboard-level refresh set
anywhere. Recommended defaults per dashboard class (set in `timeSettings`):

| Class | Example dashboards | Time range | Refresh |
|---|---|---|---|
| Status/ops | prometheus, nvidia, security, smartctl | `now-24h → now` | 30s |
| Probe/network | iperf3, smokeping, uptime, mc_monitor | `now-6h → now` | 30s |
| Trend/retention | borgmatic, navidrome, exportarr | `now-7d → now` (30d for navidrome) | 5m |
| Forecast (weather) | openweathermap | `now-24h → now+24h` | 1h |

## 8. Descriptions

**Settled 2026-09-07 — T2: titles and descriptions are both recommended, not
required.** A panel *should* carry a title (§3) and *should* carry a one-line
`description` saying what it measures and — where not obvious — what healthy looks like
or what the threshold means. Untitled chrome/status panels (exportarr's 29) are a
legitimate pattern and stay as they are. borgmatic (19/19) and nvidia (23/23) are the
model where descriptions exist: they make threshold values self-documenting. Don't
restate the metric name; say why the panel exists.

## 9. Annotations & maintenance

Folded in 2026-09-07 from Wikimedia's SRE conventions and Grafana's "iterate
continuously" guidance.

- **Annotate events that matter.** If a dashboard watches something with events —
  security: door opens/closes; borgmatic: runs and failures; the arr stack: restarts —
  configure dashboard annotations for them rather than leaving the trail only in logs.
  Keep annotation color semantics stable across dashboards: blue/cyan for manual notes,
  orange for service restarts/updates, and tag annotations with the service name.
- **Dashboards are living artifacts.** Every time a dashboard is touched, prune panels
  that no longer answer a question (renamed metrics, retired exporters, thresholds set
  for a state that no longer occurs) and refresh stale descriptions. A panel kept only
  because it was always there is debt.
- **One metric, one owner.** Don't re-create a panel that another dashboard already
  shows for the same target — link to that dashboard instead. (From the 2026-09-07
  audit: process/runtime stats belong in prometheus's Process row and
  node_exporter_full, not re-implemented in every exporter's dashboard.)

## 10. Validation checklist

Run before saving a panel/patch and after a dashboard-wide pass:

1. Structure: `panel-<n>` keys = numeric ids; no orphan/dangling elements; no overlaps;
   no `range`+`instant`.
2. Titles (where present): Title Case (T1), no whitespace edge, no unit-in-title,
   no `$var` in title; history panels end in ` Over Time`; no accidental duplicate
   titles. Untitled panels are fine if deliberate.
3. Palette: health/thresholded panels use the house scale (`green`/`orange`/`red`/
   `#EAB839`, `blue` cold-only); non-thresholded panels may use any current color; no
   `rgba(…)`, `dark-*`, or `semi-dark-*` anywhere.
4. Units: canonical IDs from §2 only; no free-text units.
5. Datasource: every query references the dashboard's `ds_*` variable — no pinned
   UIDs, no default-datasource dependence.
6. Time range/refresh match the §7 table; stat/gauge rows show the right window.
7. Every touched panel still returns one series per query (spot-check with
   `query_prometheus` instant queries before and after).
8. Purpose: dashboard `description` present; tags name the monitored services;
   non-obvious metric sources carry a legend or `links` entry (§0).
9. Display: zero-based Y (unless zoom is justified), no stray area fills, bounded
   percent graphs clamp at 100 %, no panel over ~5 series (§4).
10. No annotation-less event dashboards where events matter; no panel duplicating a
    metric another dashboard already owns (§9).

## Decisions log

| Date | Decision | Status |
|---|---|---|
| 2026-09-07 | Draft v1 written from fleet audit (14 dashboards) | Ratified same day — U1/D1/P1/P2/T1/T2 added below |
| 2026-09-07 | U1: no emoji anywhere (units, titles, descriptions); navidrome panels 40/42/43/44 → unit `none` | Settled |
| 2026-09-07 | D1: queries reference `ds_*` datasource variables, never pinned UIDs; 6 pinned dashboards + uptime migrate when touched | Settled |
| 2026-09-07 | P1/P2: color need not be uniform — house scale only where thresholds are semantically obvious (weather keeps its palette); legacy colors (`rgba(…)`, `dark-*`, `semi-dark-*`) banned everywhere, migrate on touch | Settled |
| 2026-09-07 | T1: Title Case for panel titles (function words lowercase mid-title); borgmatic + prometheus sentence-case titles migrate on touch | Settled |
| 2026-09-07 | T2: titles and descriptions both recommended, not required — exportarr's 29 untitled chrome panels stay untitled | Settled |
| 2026-09-07 | v1.1: folded in external inspiration (Grafana Labs blog, Wikimedia SRE conventions) — §0 purpose/context, §1 threshold discipline, §4 grid/graph rules, §9 annotations/maintenance | Settled |
| 2026-10-16 | Correction to §6: `hide` is an enum (`dontHide`/`hideLabel`/`hideVariable`); the guide's `hide: "variable"` was invalid and git-sync import of openweathermap-forecast-accuracy failed on it (variable-union schema conflicts). Guidance corrected to `hideVariable`. | Settled |

## Sources & inspiration

- [Getting started with Grafana: best practices to design your first dashboard (Grafana Labs)](https://grafana.com/blog/getting-started-with-grafana-best-practices-to-design-your-first-dashboard/)
- [Grafana/Best practices (Wikimedia Wikitech — SRE conventions)](https://wikitech.wikimedia.org/wiki/Grafana/Best_practices)
- [Best practices for creating dashboards (Grafana docs)](https://archive.grafana.com/docs/grafana/v7.5/best-practices/best-practices-for-creating-dashboards/)

Guidance judged org-scale or team-specific (dashboard ownership contracts, audience
personas, alerting cadence, USE/RED/golden-signals frameworks as *mandates*) was
deliberately not adopted — this fleet is single-user home monitoring, where those
reduce to "does this dashboard still tell me what I need to know."

## Open questions

None — all forks settled as of 2026-09-07 (U1, D1, P1/P2, T1, T2). New ones go here
as they come up.
