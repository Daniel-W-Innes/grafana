# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not the Grafana codebase** — it is a git mirror of dashboards from a personal Grafana instance. It contains only exported dashboard resources, folder metadata, and a LICENSE. There is no code, build system, test suite, or CI.

Single remote: `origin` — `git@github.com:Daniel-W-Innes/grafana.git` (branch `main`). This is the repo Grafana's provisioning git-sync watches (slug `repository-1cefc62`), and the only repo in the loop: user commits and Grafana write-back commits both land here. (A private Forgejo used to mirror to GitHub one-way and hid Grafana's commits from local checkouts; removed 2026-09.)

## How dashboards flow (source of truth = the live Grafana)

Grafana's provisioning git-sync imports dashboards **from the repo**, and Grafana's dashboard git integration commits **UI saves back** to the repo (author `Grafana <noreply@grafana.com>`, message `Save dashboard: <title>`). Which direction a change travels in depends on whether the dashboard already exists in Grafana:

| Situation | Editing path |
|---|---|
| Update an **existing** dashboard | MCP `update_dashboard` patch ops, applied directly to the live dashboard. Do not edit the file first — the file lags. |
| **Create a new** dashboard | Grafana refuses API-created resources in the git-synced folder (HTTP 403: "folder is managed by repo:repository-1cefc62, but the resource is not managed"). Dashboards may only *enter* from the repo: write the file here, the human commits and pushes, git-sync imports it, and only then is it editable via MCP. |

Write-back: Grafana's git integration commits dashboard saves — UI and MCP/API alike — back to `origin/main` as `Grafana <noreply@grafana.com>` commits (commit message mirrors the save/update note). Write-backs can lag or batch (observed 2026-09: a day of API edits produced no commit for hours, then a three-commit batch minutes apart). Treat a missing commit as lag, not as absence: after MCP edits, `git pull` before trusting the repo file, and compare against the live dashboard (k8s API) before hand-editing a file.

## Working with the live dashboard (MCP)

`mcp-grafana` (defined in `.mcp.json`) connects to the live instance at `http://127.0.0.1:8000/mcp`. Dashboards are v2 Kubernetes-style resources under `/apis/dashboard.grafana.app/v2/namespaces/default/dashboards/<uid>` (readable with `grafana_api_request` + a `jq` filter).

### Patch updates (targeted edits)

`update_dashboard` takes JSON-Patch-style `operations`. **Paths are relative to the dashboard `spec`** — do NOT prefix with `$.spec` (that errors with "field 'spec' is not an object"):

```jsonc
{"op": "replace", "path": "$.elements.panel-13.spec.description", "value": "..."}
{"op": "replace", "path": "$.elements.panel-13.spec.vizConfig.spec.fieldConfig.defaults", "value": {...}}
{"op": "replace", "path": "$.elements.panel-13.spec.vizConfig.group", "value": "gauge"}   // stat|gauge|timeseries
{"op": "add",     "path": "$.elements.panel-17", "value": {<full Panel element JSON>}}
{"op": "add",     "path": "$.layout.spec.rows/-", "value": {<full RowsLayoutRow JSON>}}
{"op": "remove",  "path": "$.elements.panel-5"}                                          // also delete its GridLayoutItem
```

Rules: numeric array indices only — no wildcards, no `[?(@...)]` filters; append with `/-`. Pass `uid` + a short `message` (becomes the version note).

### Panel JSON conventions (new dashboard architecture)

Read the live JSON before assembling panels — copy shapes verbatim rather than reconstructing from memory (e.g. `grafana_api_request` GET on the dashboard with `jq: '.spec.elements["panel-11"].spec.vizConfig'`).

- `metadata.name` = dashboard UID; the file name matches (`borgmatic.json` ↔ UID `borgmatic`).
- Every panel is `spec.elements["panel-<id>"]`, element key **must equal** `spec.id`. Layout `RowsLayout → RowsLayoutRow → GridLayout items` reference elements via `{"kind": "ElementReference", "name": "panel-<id>"}` — every reference must resolve (keep in sync when adding/removing panels). Do not hand-set `metadata.generation`.
- Panel data: `spec.data.spec.queries[]` are `PanelQuery`s. Queries carry `spec.query` = DataQuery with `datasource: {"name": "<datasource UID>"}` (the UID lives in the name field — current Prometheus UID is `PBFA97CFB590B2093`), `group: "prometheus"`, `editorMode: "code"`, `range: true`.
- **Never set both `"range": true` and `"instant": true`** — that is Grafana's "Both" mode and every query returns two frames, duplicating every series in the panel. House style: `range: true` only (no `instant` key at all).
- Viz groups: `stat`, `gauge`, `timeseries` (each `VizConfig` carries `version`, currently `"13.0.2"`). Gauges put `min`/`max` and absolute `thresholds.steps` in `fieldConfig.defaults`; markers on (`showThresholdMarkers: true`). House thresholds: named colors `green`/`orange`/`red` (or amber `#EAB839`), units `bytes`/`percent`/`percentunit`/`s`/`dateTimeAsIso`, `decimals: 1` for bytes.
- Selectors must pin the target explicitly, e.g. `{job="node_exporter", instance="onion.lc.brotherwolf.ca:9100", mountpoint="/run/media/daniel/stb"}`.

### Payload hygiene

Do not hand-type large JSON inline — long tool inputs get truncated and fail validation. Instead: write a small Python generator to `/tmp` (run with `nix shell nixpkgs#python3 -c python3 gen.py`), have it emit a compact single-line JSON file with structural self-checks (refs == elements, unique ids), `Read` the file, and pass its contents verbatim as typed tool parameters. Keep the generator around for regeneration.

### Verifying changes

- Read back after every save: `grafana_api_request` GET + `jq` on the elements/layout you touched.
- Sanity-run panel expressions with `query_prometheus` (instant queries) to confirm they return one series per query and sensible values.
- No image-renderer plugin is installed, so `get_panel_image` fails (HTTP 500) — visual confirmation has to come from the user in the Grafana UI.

## Commands

There is nothing to build or test. Assume a tool is missing until proven present; run it ephemerally with Nix rather than installing it. For anything scripted, use the explicit form:

```sh
# validate that a file parses
nix shell nixpkgs#jq -c jq empty <file>.json

# list dashboard titles in the current directory
nix shell nixpkgs#jq -c jq -r .spec.title *.json

# scripted edits/inspection
nix shell nixpkgs#python3 -c python3 script.py
```

Notes:
- `nix-shell -p <pkg> --run '<cmd>'` also works; `, <cmd>` (comma) is for interactive humans only — don't use it in scripts.
- These resolve against the unstable nixpkgs channel — fine for scratch tooling, but not where versions must match a flake's pinned closure.

## Conventions

- Prefer MCP over direct file edits for anything touching a dashboard that exists in Grafana (see the flow table above). When a file edit is genuinely needed (new dashboards, reconciliation), preserve the resource format exactly and keep `spec.elements`/`spec.layout` consistent.
- Repo layout mirrors Grafana folders: top-level `*.json` files are dashboards (pretty-printed JSON, 2-space indent); a folder is a subdirectory with a compact single-line `_folder.json` marker (`{"kind": "Folder", "apiVersion": "folder.grafana.app/v1", "metadata": {"name": "<folder UID>"}, "spec": {"title": "<folder name>"}}`).
- Do not run `git commit`/`git push` unless the user asks: the human commits, and Grafana makes its own write-back commits. `git pull` to sync is fine.
- Commit messages: follow the existing patterns — `Save dashboard: <title>` for dashboard updates, or a short lowercase name for a manual import (see commit `uptime`).
