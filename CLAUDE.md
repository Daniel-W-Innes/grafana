# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not the Grafana codebase** — it is a git mirror of dashboards from a personal Grafana instance, hosted on a private Forgejo (`ssh://forgejo@git.lc.brotherwolf.ca/Daniel-W-Innes/grafana.git`). It contains only exported dashboard resources, folder metadata, and a LICENSE. There is no code, build system, test suite, or CI.

## How files get here

The Grafana instance itself is a committer, via its built-in dashboard git integration. It commits changes as:

- author: `Grafana <noreply@grafana.com>`
- message: `Save dashboard: <dashboard title>`

Grafana is the source of truth: dashboards are edited in Grafana (UI or via MCP, see below) and this repo mirrors them. Hand-editing a file that Grafana may later rewrite risks conflicts — if a local edit is needed, verify no newer Grafana-side version exists and preserve the resource format exactly.

## Live Grafana access (MCP)

The `mcp-grafana` MCP server (defined in `.mcp.json`) connects to the live Grafana instance at `http://127.0.0.1:8000/mcp`. Prefer MCP tools over editing JSON files when the task involves live state: read dashboards/datasources/alerts, run queries, and apply dashboard changes directly in Grafana. Grafana then commits those changes to this repo itself — `git pull` afterwards to sync.

Repo files can lag behind the live instance. Before editing a dashboard file directly, check the live version via MCP.

## File format

Files are Grafana's new dashboard-as-code resource format (new dashboard architecture), **not** the classic export schema (`title`, `panels[]` at the top level). Each file is a Kubernetes-style resource:

- Dashboard: `apiVersion: dashboard.grafana.app/v2`, `kind: Dashboard`
  - `metadata.name` — the dashboard UID
  - `metadata.generation` — managed by Grafana
  - `spec.title`, `spec.tags`, `spec.variables`, `spec.description`, ...
  - `spec.elements` — every panel, keyed by element id like `panel-111`
  - `spec.layout` — a `RowsLayout`/`GridLayout` tree; panel placement lives here, referencing elements by name (`{"kind": "ElementReference", "name": "panel-111"}`). Repeating rows use `spec.repeat: {"mode": "variable", "value": "<var>"}`.
- Folder marker: `_folder.json` in a subdirectory, e.g. `{"kind": "Folder", "apiVersion": "folder.grafana.app/v1", "metadata": {"name": "<folder UID>"}, "spec": {"title": "<folder name>"}}` (compact single-line JSON).

Repository layout mirrors Grafana's folder structure: top-level JSON files are dashboards in the General folder; `unpoller/` is a Grafana folder (UniFi-Poller dashboards). Dashboards are pretty-printed JSON with 2-space indent; `_folder.json` is compact.

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

## Conventions for local edits

- Prefer making dashboard changes via MCP (applied live in Grafana, then mirrored here by Grafana's own commit) over editing JSON directly.
- If editing JSON directly: keep `spec.elements` and `spec.layout` consistent (every `ElementReference` must resolve to an element); do not hand-set `metadata.generation`.
- Commit messages: follow the existing patterns — `Save dashboard: <title>` for dashboard updates, or a short lowercase name for a manual import (see commit `uptime`).
