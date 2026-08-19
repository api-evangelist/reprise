---
name: reprise-injection
description: Reprise Data Injection configuration workflow (v2). MUST be invoked before any `injection_*` MCP call. Primarily for injecting data into a LIVE application via the Reprise extension; secondarily into a Reprise clone. Covers populating charts, tables, KPI tiles, dropdowns by swapping API responses at runtime. Triggers on any `injection_*` MCP tool, `dataset_id`, `Dataset → Source → Value`, `data injection`, or `Data Studio`. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct keyword matches. Body has the v2 atomic-tool surface, the two-phase wiring/data framework, the load-bearing principles, the canonical entry point, and the top catastrophic gotchas. Full vocabulary lives in `injection_docs(slug='injection')`; response-template derivation in `injection_docs(slug='injection-authoring')`; the matching-rules catalog in `injection_docs(slug='injection-conditions')`; adapter mechanics in `injection_docs(slug='injection-adapters')`; the activation/handshake reason taxonomy in `injection_docs(slug='injection-play')`. NOT a target — data injection cannot target a Product Tour directly.
version: 0.1.1
---

# Reprise Data Injection (v2)

Data Injection swaps specific web request responses on a target site with structured data, so charts / tables / KPI tiles / dropdowns render with the data the demo needs.

## Targets, in priority order

1. **Live application + Reprise extension** (primary). The user loads the live site with the Reprise extension (or the legacy Reprise Builder: Data Injection extension) installed; injected rules intercept matching requests at runtime.
2. **Reprise clone** (secondary). Same rules can fire inside a clone preview when configured.
3. **NOT a target — Product Tours.** To get injected data into a tour, inject into the live app first, then capture the live, injected app via the tour capture flow.

## Requirements

- **Browser-automation MCP** launching **real Chrome** (not bundled Chromium — extension installs need real Chrome).
- **The Reprise extension** installed in that Chrome, from the extensions page (`https://app.getreprise.com/extensions/`; sign in to Reprise first). The legacy Reprise Builder: Data Injection extension is still accepted as a fallback. Without either, `injection_dataset_play(state='play')` returns `extension_not_installed`.

## v2 atomic tool surface (21 tools)

The injection surface is 20 atomic data tools across three domains, plus `injection_docs` (the guide entry point) — 21 total:

| Domain | Tools |
|---|---|
| **Dataset** | `injection_dataset_list`, `injection_dataset_get`, `injection_dataset_create`, `injection_dataset_add_sources`, `injection_dataset_duplicate`, `injection_dataset_update`, `injection_dataset_delete`, `injection_dataset_play(state='play'\|'stop')`, `injection_dataset_export`, `injection_dataset_export_status`, `injection_dataset_import` |
| **Source** | `injection_source_list`, `injection_source_get`, `injection_source_update`, `injection_source_delete`, `injection_source_value_get`, `injection_source_value_set` |
| **Dataset Adapter** | `injection_dataset_adapter_get`, `injection_dataset_adapter_set`, `injection_dataset_adapter_clear` |

Sources are minted via `injection_dataset_create(sources_json=...)` or `injection_dataset_add_sources(...)` — there's no standalone source create. Each dataset has at most one active adapter (1:1 in the product UI); the three `injection_dataset_adapter_*` tools cover read/upsert/clear.

`injection_dataset_play(state='play'|'stop')` toggles injection on/off. `injection_dataset_export(dataset_id=...)` kicks an async export and returns a task handle — poll `injection_dataset_export_status(dataset_id=...)` until `ready`/`failed`. `injection_dataset_import(file_path=...)` is one atomic call. (Neither takes a `wait` arg.)

## The two-phase framework — don't mix them

Two loops with distinct failure axes. Mixing them is the #1 wasted-time pattern.

- **Phase A — Plumbing.** Wire up source + adapter + response_template with **sentinel values** (`99999`, `"DUMMY"`, dates `1999-01-01..30`). Failure means wiring is wrong.
- **Phase B — Data.** Replace sentinels with the user's intended data. Failure means values, not wiring.

**Key principle: data lives in the Value, nowhere else.** Don't put data values in the adapter (its `parse_function` / `serialize_function` are pure transforms), in the source's `conditions` (they match requests, not data), or via browser-automation typing (bypasses Reprise entirely; the next reload reverts). If you find yourself typing values somewhere other than the Value, you're off-track.

## Vocabulary

Data Studio uses a flat **Dataset → Source → Value** model. A Source is one intercepted endpoint — it carries the `target_url`, `conditions`, `response_template`, an optional adapter link, and its data rows (Values). Full vocabulary at `injection_docs(slug='injection')`.

## Workflow

**Phase A — Plumbing with sentinels:**

1. **Identify the target request** via the BMA. Record URL + method + raw response body. Note `fetch` vs `XMLHttpRequest` (adapters only intercept `fetch`).
2. **Patterns.** `injection_docs(slug='injection-patterns')` — the data-injection pattern corpus (TOC with excerpts by default; pass `section='<name>'` for one topic's full text).
3. **Derive the schema from the captured body** — don't invent it. Either pass the raw body in `response_template` (inference produces the triple) or pass the `{template, data_map, injection_point}` triple under `response_template_raw` when inference can't cover the shape. Full derivation algorithm at `injection_docs(slug='injection-authoring')`.
4. **Build the dataset + source + Value in ONE atomic call.** See "Canonical entry point" below.
5. **Open the dataset editor** at `https://<host>/data-studio/datasets/<dataset_id>` in a side tab — ground truth for what Reprise has stored. Keep open through Phase B to disambiguate wiring vs data failures.
6. **Play.** `injection_dataset_play(dataset_id=..., state='play')` returns a `handshake_expression` — run it via the BMA. Structured `{success, reason}` reply distinguishes real success from `extension_not_installed` / `extension_outdated` / `extension_not_responding`.
7. **Verify sentinels render.** Sentinels visible → Phase A passes. Originals still visible → plumbing fails. Page blank → injection over-replaced.
8. **Iterate plumbing only** on miss via `injection_source_update(...)`, then `injection_dataset_play(state='stop')` + `injection_dataset_play(state='play')`. Don't touch data values here.

**Phase B — Real data:**

9. **Re-confirm data shape.** Rows-not-columns (30-day chart = 30 rows × few columns). Scalar cells only. Display Names in `data_map` must match the keys you're about to write.
10. **Replace sentinels** via `injection_source_value_set(dataset_id=..., source_id=..., value_id=..., data='[...]')`, replay, glance at the editor tab to confirm rows updated.
11. **Verify against intent.** Editor matches intent but page doesn't → wiring (back to Phase A). Editor rows wrong → data (back to step 10).
12. **Hand off.** Share the editor URL with the user so they can keep iterating.

## Canonical entry point — one atomic call

**`injection_dataset_create(name=..., sources_json=...)`** creates the dataset + every source + every Value in a single transaction. This is the only entry point for minting Sources alongside the dataset — there's no standalone source create in v2.

```json
[
  {
    "section": "<group name>",
    "name": "<source name>",
    "target_url": "<step-1 URL>",
    "conditions": [
      {"type": "method", "value": "<step-1 method>"},
      {"type": "url", "test": "contains", "value": "<step-1 path fragment>"}
    ],
    "response_template": <raw body from step 1>,
    "metadata": {"sync": true},
    "values": [
      {"name": "sentinel", "data": [{"type": "replace", "entities": [<sentinel rows>]}]}
    ]
  }
]
```

Sentinels keyed by Display Names from the inferred `data_map`. Mismatched keys = empty cells. Multiple sources / values: more entries — still one atomic call. Pass raw body in `response_template`; pass the triple in `response_template_raw` (backend rejects the triple in `response_template` at preflight).

## Adapter — 1:1 dataset-scoped

Each dataset has at most one active adapter. Three tools:

- `injection_dataset_adapter_get(dataset_id=...)` — read the active adapter (full code body + `adapter_config`).
- `injection_dataset_adapter_set(dataset_id=..., parse_function=..., serialize_function=..., adapter_config='{}')` — upsert. Creates on first call, PATCHes on subsequent. **PATCH preserves what you don't pass:** omit `adapter_config` and the previous value is retained; omit `parse_function` and the previous body is retained. The 1:1 invariant is enforced server-side (sets `always_run=True`, clears sibling rows).
- `injection_dataset_adapter_clear(dataset_id=...)` — unlink the active adapter. Idempotent.

There are no standalone adapter-library tools — the surface is opinionated for the 1:1 dataset↔adapter relationship the product enforces.

## Top catastrophic gotchas

The full catalog is in `injection_docs(slug='injection-authoring')` (matching rules in `injection_docs(slug='injection-conditions')`). These three are silent — accepted at create time, break at runtime, no error:

1. **`metadata.sync = true` or the rule never fires.** Without it, the rule isn't included in `play_config.sync_interception_rules` and the runtime filter treats it as inactive.
2. **Conditions shape — typed-dict form is mandatory.** Flat shorthand (`{"method": "GET"}`) is accepted at create but **never matches at runtime** — the matcher reads `rule.type` / `rule.value`. Always: `{"type": "method", "value": "GET"}`, `{"type": "url", "test": "contains", "value": "/api"}`.
3. **One source per `(target_url, injection_point)` pair — last `replace` wins.** N sources sharing the same target URL + array `injection_point` have `apply_rule_modifications` walk them in order; the last `replace` wins and the others' values are silently overwritten. One response feeding N charts → ONE source whose `data_map` has N columns.

## After every session

`platform_summary_report` + `platform_friction_report` per distinct issue — see `reprise-session-close`.

For everything else: vocabulary and the data model in `injection_docs(slug='injection')`; the response-template derivation algorithm, response-shape trims, and resuming prior work in `injection_docs(slug='injection-authoring')`; the matching-rules reference in `injection_docs(slug='injection-conditions')`; adapter mechanics in `injection_docs(slug='injection-adapters')`; the activation/handshake reason taxonomy and export/import in `injection_docs(slug='injection-play')`.
