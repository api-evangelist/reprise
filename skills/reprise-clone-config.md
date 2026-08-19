---
name: reprise-clone-config
description: Reprise Clone configuration workflow (v2). MUST be invoked before any action on a Reprise clone — writing or editing snippets, diagnosing white screens / blank panels / stale data on a clone preview, routing API endpoints, or walking a capture path. Triggers on `clone_id`, any `clone_*` MCP tool, `/sw-setup/`, `clone_api`, `replay_backend`, or any clone debug/preview URL. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct keyword matches. Body has the v2 atomic-tool surface, the snippet-type vocabulary, the phase outline, the two forcing functions before any snippet, the two-stage routing model, and the top catastrophic gotchas. Telemetry blocklist, full gotcha catalog, state-management chooser, visual-symptoms triage, hallucinated-method list, snippet timing slots, and `clone_api` method reference all live in `clone_docs(slug='clone')` and `clone_docs(slug='clone-api')`.
version: 0.1.0
---

# Reprise Clone Configuration (v2)

A Reprise Clone is a captured, interactive replica of a web app served by a service worker from cached network responses. Configuring a clone means making the captured content work correctly when served from the clone.

The MCP doesn't drive a browser; it reads + mutates the Reprise-side clone object that the **Reprise Builder: Clone** Chrome extension wrote to during capture, plus inspects the SW's runtime via the debug surface.

The v2 Clone surface is atomic — one verb per tool (`clone_get`, `clone_request_analyze`, `clone_snippet_create`, `clone_debug_start`, …) rather than the v1 `clone(action='…')` dispatch. There is **no** `clone_publish` on this surface, and `clone_analytics` is staff-only (hidden from the catalog unless you're authenticated as staff). Don't call a tool you can't see in the catalog.

## Snippet types

`clone_snippet_create` (and `clone_snippet_update`) take a `type`:

- **`backend`** — JS that runs **inside the service worker**. Use for request matching, routing, and response transformation. The right tool for almost every clone-fix.
- **`custom_js`** — JS that runs **in the page context** (DOM patches, UI tweaks). **Cannot fix SW-level request issues.**
- **`websocket`** — WebSocket handler.

Legacy codes `RB` / `C` / `WS` are still accepted but new code should use the canonical names. The SW global is `clone_api` (also exposed as `replay_backend`; both names point to the same instance — don't rewrite existing snippets between them as a side effect of unrelated edits).

## Two forcing functions — do these BEFORE writing any snippet

Skip either and you will write broken snippets.

1. **`clone_docs(slug='clone-api')`** — authoritative method reference for the SW global. **Never guess a method name.** Calling a non-existent method throws at snippet-init and **crashes the entire service worker**, taking the clone down. If the reference doesn't list what you need, fall back to `match_domain().pre_process()` / `.handle()` and build the behavior explicitly.
2. **`clone_docs(slug='clone-patterns')`** — the clone pattern corpus (snippet recipes, RB API quirks, request-matching / routing fixes, known gotchas), read in full rather than keyword-searched. The bare call returns a table of contents with excerpts; pass `section='<name>'` to pull one topic's full text.

## Workflow

1. **Assess.** `clone_get(clone_id)` → notes + existing snippets; `clone_request_analyze(clone_id)` → routing candidates; `clone_path_get(clone_id, include_health=True)` → content health; `clone_request_duplicates(clone_id)` → plan `match_id` snippets using the `recommended_id` hashid (not numeric pk).
2. **Offline test** (CRITICAL). Before any URL-rewriting work, ask the user to open the preview URL with devtools Network tab on "Offline." Page renders → SW is serving from cache, focus on API routing. Page blank → check content health. A 404 in the Network panel does NOT mean the file wasn't served — the CDN plugin rewrites internally.
3. **Baseline snippets.** Block telemetry (canonical host list in `clone_docs(slug='clone')`), block original SW (`clone_api.get("/sw.js").handle(() => new Blob([""], { type: "application/javascript" }))`), `match_id` snippets for duplicates, API routing snippets for endpoints from step 1.
4. **Walk the path with the user via the debug session.** Always `clone_debug_start(clone_id)` even when the page renders fine — debug captures JS errors and failed requests the agent can't see. It returns a `session_token` + `debug_url`; open the `debug_url` first to register the SW. After each path step: `clone_debug_errors(session_token)`, then `clone_debug_errors(session_token, clear=True)` before the next step. `clone_debug_routes(session_token)` shows the registered-route table and the last 50 fetch decisions when a snippet looks right but isn't firing. Close with `clone_debug_stop(session_token)`.
5. **Apply snippet changes.** Every `clone_snippet_create` / `clone_snippet_update` response gives a `reload_via_sw_message` JS one-liner (fast path — evaluate in the preview tab) and an `apply_url` (`/sw-setup/?next=<current-path>`, fallback). Read the response's `warnings` array first — a declaration collision or byte-identical duplicate across active snippets of the same type throws at SW init and crashes every route.
6. **Verify data completeness.** A rendered page is not a working page. Row counts, placeholder cells, console errors, data-dependent interactions (sort/filter/pagination).
7. **Record.** `clone_note_create` for status + non-obvious fixes. For a reusable client technique worth adding to the shared corpus, file `platform_friction_report(category='feature_request', product='clone', ...)` with the symptom/cause/fix — the pattern corpus is repo-managed, so there's no in-session pattern write tool.

## Two-stage routing model — internalize this

Confusing these stages is the #1 source of "I added `no_hash_check()` but it didn't help" friction.

- **Stage 1 — Route match.** Decided by chain opener + path + method + optional `request_filter`. **Query string and request body are NOT inspected.** A route written as `clone_api.get("/main.js?v=:v")` never matches anything.
- **Stage 2 — Capture lookup.** Once a chain matches, which captured response does the SW serve? Decided by URL + body hash + method. Tuned by `no_hash_check()`, `payload_key_match()`, `match_id()`, `fallback_path_match()`.

If you find yourself reaching for `no_hash_check()` to fix a route that "isn't matching," you almost certainly want stage-1 plumbing instead.

## Top catastrophic gotchas

The full catalog (and the running list of hallucinated method names that crash the SW at init) is in `clone_docs(slug='clone')` and `clone_docs(slug='clone-api')`. These four are the silent or whole-clone-down failure modes worth seeing before you write a single snippet:

1. **One broken `backend` snippet crashes ALL routes.** Any backend snippet that throws at init breaks every other snippet. Errors only surface in the browser's SW devtools console, not MCP debug output. If everything suddenly stops working, check the most recently created/updated snippet first.
2. **`handle()` always fires when the route matches — no cache-miss fallthrough.** It hijacks every matching request. Scope with a `request_filter` callback on the chain opener; use `post_process()` to modify a cached response.
3. **Path routes are same-origin only.** `clone_api.get("/api/data")` only matches the clone's own domain. For cross-origin APIs use `clone_api.match_domain("api.example.com")`.
4. **GraphQL `handle()` on `/graphql` breaks all cached queries.** Use a POST body filter on the chain opener to scope to a single operation.

## After every session

`clone_session_end(clone_id, summary='…')` checkpoints the clone's edit session so the work shows grouped in the Edit Sessions tab — skip it and the backend's idle-timer closes the session after ~5 min. Then `platform_summary_report` once + `platform_friction_report` once per distinct issue — see `reprise-session-close`.

For everything else — full gotcha catalog, hallucinated-method list, telemetry blocklist, visual-symptoms triage, state-management chooser, POST-routing patterns, third-party iframe handling, snippet timing slots, `clone_api` method reference — `clone_docs(slug='clone')` and `clone_docs(slug='clone-api')`.
