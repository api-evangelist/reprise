---
name: reprise-tour-id-model
description: Reprise Product Tour ID-kind resolution (v2) — three IDs (`draft_id`, `published_id`, `PublishedReplayLink.id` from `/launch/<slug>/`), no auto-flip, `tour_get(...)` is the canonical discovery call. MUST be invoked whenever an ID-related error surfaces or the user pastes a launch URL. Triggers on `wrong_id_kind`, `tour_not_found`, `exactly_one_id_required`, `missing_draft_id`, `missing_published_id`, `/launch/<slug>/`, "which ID do I pass", "is this a draft or published", or a paste of any opaque tour ID where the kind is unclear. Usually invoked by the `reprise-mcp` router; can also auto-fire on direct error / launch-URL matches. Body has the three ID kinds, per-tool ID requirements at a glance, the `tour_get(...)` discovery pattern, and the common error shapes. Full error taxonomy with hint text + wire-format mapping + end-to-end worked example live in `tour_docs(slug='tour')`.
version: 0.1.0
---

# Reprise Tour ID Model (v2)

Tours have three ID kinds; tools accept only the kind(s) they operate on; there is **no auto-flip** between them. If you have the wrong kind, call `tour_get(...)` first to discover the labeled pair.

## The three kinds

| Kind | What it is | Exists when |
|---|---|---|
| `draft_id` | The authoring tour. Always present. | Always — every tour starts as a draft |
| `published_id` | The snapshot served at preview URLs. | Only after `tour_publish(draft_id=...)` |
| `PublishedReplayLink.id` | The `<slug>` in `https://<host>/launch/<slug>/`. A *third* kind — not a draft or published id. | Once a published tour has at least one shareable link |

A brand-new tour has only `draft_id`. After publish: both. A tour with unpublished edits: both, but the draft is newer than the published snapshot. A link slug from a `/launch/<slug>/` URL is neither — it's a `PublishedReplayLink.id` that maps to one specific published tour (and through that, to a draft).

## Per-tool ID requirements at a glance

| Bucket | Tools | Param signature |
|---|---|---|
| Strict draft | `tour_guide_*` (all); `tour_publish`; `tour_injection_get`, `tour_injection_set`; `tour_dom_*`; `tour_variable_*`; `tour_screen_copy` (source + target draft) | `draft_id` only |
| Strict published | `tour_link_*` (all) | `published_id` only |
| Either, exactly-one-required | `tour_get`; `tour_screen_list`, `tour_screen_search`; `tour_duplicate`, `tour_archive`, `tour_restore`, `tour_move`, `tour_metadata` | `draft_id` or `published_id`, exactly one |

`tour_search` and `tour_list` take no input ID — the search index is keyed by `draft_id` (the backend indexes draft tours).

## Discovery: `tour_get(...)` is how you get the other ID

`tour_get(...)` is the universal lookup. Pass whichever ID you have under either `draft_id` or `published_id`; the response includes both labeled fields. A `PublishedReplayLink.id` (the `/launch/<slug>/` slug) resolves transparently — pass it as `draft_id` or `published_id` and the call echoes `link_id` alongside the labeled pair.

```
have published_id, need draft (e.g. to call tour_guide_create):
  → tour_get(published_id="pub-X")
  ← {draft_id: "draft-Y", published_id: "pub-X", ...}

have a /launch/<slug>/ URL (PublishedReplayLink.id — third kind):
  → tour_get(draft_id="kX0EKvy")           # pass under either arg
  ← {draft_id: "draft-Z", published_id: "pub-Z", link_id: "kX0EKvy", ...}
```

Other entry points that surface labeled IDs without one in hand: `tour_list()` (no args), `tour_search(query=...)`, and `tour_publish(...)` (returns the new `published_id`).

Every success response on a `tour_*` tool includes both `draft_id` and `published_id` at the top level (`published_id` is `null` when the tour was never published). The wire-format still uses `replay_id` internally; the v2 surface translates to `draft_id` / `published_id` at the tool boundary.

## Common errors and what they mean

- `wrong_id_kind` — you passed the wrong kind for this tool. Hint includes which kind to pass and how to discover it via `tour_get(...)`.
- `tour_not_found` — the value didn't resolve as the kind you passed AND didn't resolve as a `PublishedReplayLink.id` for your client either. Verify the kind matches; common cause is pasting a `<slug>` from a `/launch/` URL while expecting it to be a draft id.
- `exactly_one_id_required` — you passed both `draft_id` and `published_id` to an either-tool. Pass exactly one.
- `missing_draft_id` / `missing_published_id` — strict-kind tool called with no ID or the wrong kind.

Full hint text for each error, the wire-format mapping (`demo_kind="replay_draft|replay_published"` is still what the backend stores internally), and a worked end-to-end example are in `tour_docs(slug='tour')`.

## After every session

`platform_summary_report` + `platform_friction_report` per distinct issue — see `reprise-session-close`.

For new-tour capture: `reprise-tour-capture`. For editing an existing tour: `reprise-tour-edit`.
