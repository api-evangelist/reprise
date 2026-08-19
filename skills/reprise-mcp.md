---
name: reprise-mcp
description: Required entry point and router for ALL Reprise MCP v2 work. MUST be invoked immediately whenever the user wants to do anything with Reprise — fixing / building / editing / debugging / rebranding a Reprise demo, tour, clone, or captured app; troubleshooting blank or broken pages in a Reprise context; or any mention of "Reprise", "demo" (in a Reprise context), "tour", "capture", "preview URL", or any Reprise MCP tool. Users describe tasks in plain English ("fix my demo", "add data to this page", "build a tour of your app", "make this look like a different prospect") without naming Reprise's product surfaces; this router maps natural-language asks to the right surface skill (`reprise-clone-config`, `reprise-tour-capture`, `reprise-tour-edit`, `reprise-tour-id-model`, `reprise-injection`, or `reprise-session-close`) and invokes it explicitly. Always read this skill first on any Reprise turn — it is the entry point. This is the v2 router; v2 exposes atomic per-action tool names (e.g. `tour_dom_text_edit`, `injection_dataset_create`) and tools live under `/v2/mcp/` and per-product scoped paths.
version: 0.1.1
---

# Reprise MCP v2 — Router

You've landed here because the user wants to do Reprise work. This skill carries **no workflow content** — its only job is to identify the right surface and hand off to the surface-specific skill. Don't try to do the work yourself from this skill; route correctly and the right skill will load with everything you need.

## Step 1 — Confirm Reprise v2 context

You're in scope if:
- The user mentions Reprise, a Reprise preview URL, a Reprise demo, a tour, or any Reprise MCP tool.
- The Reprise MCP v2 is connected. Tools you'll see: `tour_*` (61 atomic tools, incl. `tour_docs` / `tour_search`), `injection_*` (21 atomic tools, incl. `injection_docs`), `clone_*` (atomic tools, incl. `clone_docs` for Clone product guides), and cross-product `platform_*` (9): `platform_whoami`, `platform_friction_report`, `platform_friction_status`, `platform_summary_report`, plus the shared asset library `platform_folder_*` / `platform_asset_move`. Per-product scoped endpoints (`/v2/mcp/tour/`, `/v2/mcp/injection/`, `/v2/mcp/clone/`) expose a smaller catalog for callers who only need one product.

If neither is true, this skill shouldn't have fired — answer the user's actual question without routing.

These skills target the **atomic** Reprise MCP surface (per-action tools like `tour_dom_text_edit`). If you instead see only **polymorphic** tools (single tools like `tour_edit`, `tour_capture`, `tour_lifecycle`, `injection_widget`, `html_*`), your client is on the legacy MCP surface — reconnect it to the current Reprise MCP endpoint so the atomic tools these skills drive are available.

## Step 2 — Identify the surface from cues

| Surface | The user is asking for | Cues |
|---|---|---|
| **Clone** | Fix / debug / configure a captured interactive app | `clone_id`, snippet, `clone_api`, `replay_backend`, any `clone_*` MCP tool, `/sw-setup/` URL, "captured app", "the demo white-screens / goes blank / breaks after login", service-worker issues, snippet editing |
| **Product Tour, NEW capture** | Record a new tour by capturing screens | `tour_capture_*`, `pairing_token`, `deep_link_url`, `tour_capture_now`, a brand-new `draft_id`, "Reprise extension", "record a tour", "build a tour of X", "capture screens" |
| **Product Tour, EDITING existing** | Modify an existing tour (text, attributes, images, guides, re-skin) | `tour_dom_*`, `tour_screen_copy`, `tour_guide_*`, `tour_variable_*`, `tour_link_*`, `tour_injection_set`, "edit this tour", "re-skin for prospect Y", "rebrand the tour", "add a guide / variable" |
| **Product Tour, ID confusion** | Resolve a paste of a `/launch/<slug>/` URL or a `wrong_id_kind` error | `draft_id`, `published_id`, `PublishedReplayLink`, `/launch/<slug>/`, `wrong_id_kind`, `tour_not_found`, "what's the draft for this published tour", "I have a launch URL" |
| **Data Injection** | Populate charts / tables / dropdowns with custom data by swapping API responses | any `injection_*` MCP tool, `dataset_id`, `Dataset → Source → Value`, "Data Studio", "populate the chart", "fake data on this page", "swap the API response", "inject data into the live app" (primary case) |

If exactly one surface matches, go to Step 4.

## Step 3 — Ask if ambiguous

If the prompt doesn't clearly indicate which surface (common with vague phrasings like "fix my Reprise demo," "build me a demo for [prospect]," "the page is broken"), ask **one** clarifying question:

> Are you working on:
> - **Clone** — an interactive captured app you're configuring or debugging?
> - **Product Tour** — a linear walkthrough from captured screens? *(And: building new, or editing existing?)*
> - **Data Injection** — swapping API responses on a live app to populate charts / tables?

Don't guess; don't proceed without the answer. A wrong surface guess wastes far more time than one clarifying turn.

## Step 4 — Invoke the surface skill

Call the right skill explicitly. Do this BEFORE any other tool call:

- **Clone** → `Skill(skill="reprise-mcp-skills:reprise-clone-config")`
- **Tour, new capture** → `Skill(skill="reprise-mcp-skills:reprise-tour-capture")`
- **Tour, editing existing** → `Skill(skill="reprise-mcp-skills:reprise-tour-edit")`
- **Tour ID confusion** → `Skill(skill="reprise-mcp-skills:reprise-tour-id-model")`
- **Data Injection** → `Skill(skill="reprise-mcp-skills:reprise-injection")`

For Product Tour, the verb is the signal: "build / record / capture / make a tour" → `reprise-tour-capture`; "edit / re-skin / change / fix / add a guide / rebrand" → `reprise-tour-edit`. The ID-model skill is a focused side-load when you hit an ID error or a launch-URL paste.

## Step 5 — At end of session

Regardless of which surface skill ran, when the user signals end-of-session ("we're good", "that's it", "thanks", topic switch) or when you've completed the task and are composing a final summary, it's recommended to invoke:

`Skill(skill="reprise-mcp-skills:reprise-session-close")`

That skill carries the `platform_summary_report` and `platform_friction_report` call shapes.

## What this skill is NOT

- It's not a workflow skill. Don't enumerate phases from here.
- It's not the place for gotchas, forcing functions, or method references — those live in the surface skills' bodies and in the matching `*_docs(slug='...')` for the product you're working in (`tour_docs`, `injection_docs`, `clone_docs`).
- It's not a fallback if you forgot which skill to invoke — pick one consciously.
