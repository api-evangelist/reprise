---
name: reprise-tour-capture
description: "Reprise Product Tour capture workflow (v2) — recording NEW tours via the Reprise Chrome extension. MUST be invoked before any `tour_capture_*` call. Triggers on `tour_capture_session_start`, `tour_capture_now`, `pairing_token`, `deep_link_url`, a brand-new `draft_id`, or the Reprise extension. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct keyword matches. Body has the 7-step flow at headline depth, the 3-tier extension-pairing escalation, the load-bearing page-settle / verify-the-page principles, and operational gotchas around browser-automation MCPs and proxy timeouts. Per-tier mechanics, `tour_capture_now` envelope, post-stop cleanup, and install URLs live in `tour_docs(slug='tour-capture')`; theming in `tour_docs(slug='tour-theming')`; re-skinning and composing in `tour_docs(slug='tour-authoring')`. NOT for editing existing tours — use `reprise-tour-edit`."
version: 0.1.0
---

# Reprise Product Tour Capture (v2)

A Product Tour is a linear walkthrough built from captured DOM snapshots. Capture happens in the user's own Chrome via the **Reprise** extension — the agent doesn't snapshot pages itself; it orchestrates.

## Preload tool schemas before starting

If your client uses deferred tool schemas (Claude Cowork, etc.), preload everything you'll need in **one** `ToolSearch` call before step 1 — discovering tools mid-flow forces back-to-back schema loads that fragment the workflow. The full set for a capture session:

- Reprise MCP (v2): `tour_create`, `tour_publish`, `tour_capture_session_start`, `tour_capture_session_status`, `tour_capture_session_stop`, `tour_capture_toolbar_mount`, `tour_capture_now`, `tour_screen_list`, `tour_screen_get`, `tour_screen_node_list`, `tour_guide_create`, `tour_docs` (for install-URL fallback), `platform_whoami` (only if you have multiple Reprise MCPs connected and need to pick by host)
- Browser-automation MCP (Claude in Chrome, Playwright, etc.): `list_connected_browsers`, `select_browser`, `tabs_context_mcp`, `navigate`, `javascript_tool` (and your tool inspector / `get_page_text` equivalent for verifying iframe presence)
- Task tracking: `TaskCreate`, `TaskUpdate` (if your client has them and the flow is ≥3 steps)

## The capture flow

1. `tour_create(title=...)` → `{draft_id, preview_url}`.
2. `tour_capture_session_start(draft_id=...)` → `{pairing_token, deep_link_url, ...}`.
3. **Pair the extension** (3-tier escalation, see below — highest failure point).
4. `tour_capture_session_status(pairing_token=...)` until `extension_last_seen_at` is fresh (<10s) ⇒ paired.
5. **On browser-automation MCPs** (Claude in Chrome, Playwright, etc.): first `navigate(tabId, target_url)` to put the MCP tab onto the URL, *then* `tour_capture_toolbar_mount(pairing_token=..., target_url='https://...')`, *then* probe the DOM for `#reprise_iframe` to confirm — see "Browser-automation MCPs" below. **On user-Chrome drivers** (AppleScript, native browser-use): `tour_capture_toolbar_mount(...)` alone is enough; the SW finds the user's tab by URL and reloads it.
6. Per-screen loop: browser MCP navigate / click → wait for settle → `tour_capture_now(pairing_token=..., title='Step N', target_url='...', wait=False)` → poll `tour_capture_session_status(pairing_token=...)` and **verify `screen_count` actually incremented**. `tour_capture_now` returns `ok=true` even when no tab matches `target_url` — the SW accepted the command, but couldn't dispatch it; nothing was captured. `screen_count` unchanged + `extension_last_seen_at` still fresh = the toolbar isn't mounted in any tab the SW can find. Recovery: re-`navigate(tabId, target_url)` first, then re-issue `tour_capture_toolbar_mount` — and because the same `(action, payload)` is deduplicated server-side for ~120s, append a cache-buster (`target_url='https://example.com/?_rt=1'`) on the retry so the command actually reaches the SW. **Default to `wait=False`.** The default `wait=True` sync window exceeds the ~25-30s cap on the Anthropic MCP proxy and Claude Cowork — the underlying capture completes but the response gets killed mid-flight, which looks like a failure when it isn't.
7. `tour_capture_session_stop(pairing_token=...)`, then `tour_publish(draft_id=...)`.

**Always pass `target_url` to `tour_capture_toolbar_mount` and `tour_capture_now`.** Without it the SW falls back to "active tab in the last-focused Chrome window" and silently picks the wrong tab in multi-window or background-agent setups.

## Pairing the extension — 3-tier escalation

The `deep_link_url` is a `chrome-extension://...` URL. Browser-automation MCPs can't `navigate` to those (CDP rejects them as browser-internal). Try in order, escalate on failure:

- **Tier 1 — JS-assignment from a real-page tab** (fully automated). From a tab on a real `https://` page, evaluate `window.location.href = '<deep_link_url>'` as **its own tool call** (don't chain with a preceding navigate — they race). If your browser-MCP nudges you toward `browser_batch` to combine calls, ignore it for the pairing flow — the JS-assign must be its own call. **Authoritative pairing signal:** poll `tour_capture_session_status(pairing_token=...)` for ~10s after the assignment; `extension_last_seen_at` non-null and fresh ⇒ paired. The JS step itself returns nothing informative — a stranded-tab error after the assign is a positive secondary signal *when it appears*, but absence ≠ failure. **Fast diagnostic for extension-not-installed:** if your browser MCP's tab inspector shows the JS-assigned tab URL as `chrome-extension://invalid/`, the extension isn't installed — skip the retry loop and go straight to `AskUserQuestion` + the install URL from `tour_docs(slug='tour-capture')`. Otherwise, if status stays null past ~10s, ask the user whether the extension is installed before retrying.
- **Tier 2 — User pastes the deep link** into Chrome's address bar.
- **Tier 3 — User pastes the token** into the popup's "Pair with agent" field.

Never send the `chrome-extension://` URL as a clickable `<a href>` — Chrome blocks the click. Full per-tier mechanics, race-condition details, and stranded-tab recovery in `tour_docs(slug='tour-capture')`.

Don't proceed past `tour_capture_toolbar_mount` until status confirms pairing — commands to an unpaired session silently no-op. If the user doesn't have the extension installed, send them to the install URL in `tour_docs(slug='tour-capture')`.

## Browser-automation MCPs (Claude in Chrome, Playwright, etc.)

The extension's `tour_capture_toolbar_mount` allow-lists the host and reloads any tab whose URL matches `target_url` (it queries Chrome for all tabs on that origin — automation tab groups included; that part isn't the issue). The catch: **on browser-automation MCPs the agent's tab is typically NOT yet on `target_url` when `tour_capture_toolbar_mount` is called**, so the SW finds no tab to reload and the toolbar never mounts. Re-issuing the same `tour_capture_toolbar_mount(target_url=...)` within ~120s is deduplicated server-side, so the retry silently no-ops.

Correct order on a browser-automation MCP:

1. `navigate(tabId, target_url)` — put the MCP tab onto the target URL first. Wait for settle.
2. `tour_capture_toolbar_mount(pairing_token=..., target_url=...)` — the SW now finds the tab and reloads it; the content script injects.
3. After settle, probe the DOM for `#reprise_iframe`. Its presence is the *only* signal that the toolbar mounted.
4. If absent: verify `tour_capture_session_status` shows the extension as fresh, then re-`navigate` and re-issue `tour_capture_toolbar_mount` — but append a cache-buster (`?_rt=1`) to the `target_url` on the retry so it doesn't dedup against the prior call.

This only matters for the first `tour_capture_now` in the session. Subsequent same-origin navigations keep the iframe mounted.

On user-Chrome drivers (AppleScript, native browser-use) you don't need the upfront navigate: the user's tab is already on a real page, the SW reloads it, and the content script mounts. Skip straight to `tour_capture_toolbar_mount`.

## Two pre-capture principles that prevent the most common silent failures

1. **Page-settle before each `tour_capture_now`.** A capture of an unsettled page produces an unstyled / half-rendered screen with **no error signal**. Between navigation and `tour_capture_now`: `page.waitForLoadState('networkidle')`, or `browser_wait_for` on a sentinel hero-text element, or an 800–1500ms blanket sleep. Re-apply after every cross-document navigation.
2. **Verify the page is what you intended.** `tour_capture_now` happily snapshots a 404, a soft-redirect, a "you must accept cookies" interstitial, or a CTA-triggered overlay — none fail your call. Before capturing: confirm URL, confirm title isn't a 404, confirm a sentinel element from the intended content is present. After CTA clicks (booking, signup, pricing), screenshot to verify; if an overlay landed on top, dismiss it before `tour_capture_now`.

## Preview before publish

`tour_capture_now` returns `flush_complete=true` when every screen is processed. **Always wait for `flush_complete=true` before opening `preview_url`** — otherwise the preview shows placeholders for screens still being processed.

## Adding guides during capture

When the user asks for guides on the captured screens, **default to tethered (pointing) guides, not floating**. A floating guide is a card hovering in the middle of the screen with no anchor — it works for intro/outro/banner messages but it's the wrong default for "explain this feature." Pointing guides attach to a real DOM node and visually call out what they describe.

Per-screen flow:

1. `tour_screen_node_list(screen_id=<from tour_screen_list>)` — returns up to 200 anchorable candidates (interactive elements, headings, aria-labeled nodes) with `sel` / `text` / `role` / `aria_label`. Pick the node whose `text` or `aria_label` matches the feature the guide describes.
2. `tour_guide_create(draft_id=..., screen_id=..., guide_type='tethered', target_node_id=<from step 1>, text='<h2>Title</h2><p>Body</p>', buttons='[{"text":"Next","action":"flow_next_screen"}]')`. The `position` field auto-defaults sensibly (the server walks ancestors for header/nav and places the popup below if found, above otherwise) — only override when the default clips off-viewport.

Only fall back to `guide_type='floating'` when there's genuinely no good anchor: a welcome screen, an outro CTA on an empty/transition screen, or a callout that's about the whole screen rather than one element. In that case, pass only the floating-compatible fields — `position` / `target_node_id` / `element_click` / `caret_enabled=True` / `pulse_enabled=True` are rejected at the MCP boundary with `floating_with_<field>` errors.

End-of-tour CTAs: the last guide on the last screen should use `restart_demo` or `goto_url` rather than `flow_next` / `flow_next_screen` (those silently no-op when there's nothing to advance to).

## Transient errors

Reprise MCP calls occasionally return `Session terminated` from the transport layer (not the server). **Retry once before assuming the session is dead** — most return cleanly on the second attempt.

## After every session

`platform_summary_report` + `platform_friction_report` per distinct issue — see `reprise-session-close`.

`tour_capture_now` envelope fields, async / fire-and-forget mode, post-stop toolbar cleanup, and install URLs are in `tour_docs(slug='tour-capture')`; theming via `--rguide-*` in `tour_docs(slug='tour-theming')`; re-skinning and composing tours from screens of other tours in `tour_docs(slug='tour-authoring')`. For editing existing tours (not capturing new ones), use the `reprise-tour-edit` skill. If you have a `/launch/<slug>/` URL or hit a `wrong_id_kind` error, see `reprise-tour-id-model`.
