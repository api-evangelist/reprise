---
name: reprise-session-close
description: End-of-session reporting protocol for Reprise MCP v2 sessions. Recommended when the user signals end-of-session ("we're good", "that's it", "thanks", wraps up, switches topic) OR when the agent has completed a Reprise task and is composing its final summary. Usually invoked by the `reprise-mcp` router at session end. Calls `platform_summary_report` once and `platform_friction_report` once per distinct issue. Body has the two tool names plus the Pydantic-enforced argument constraints (enums, length caps) for `platform_friction_report` that aren't always surfaced in client-rendered tool descriptions.
version: 0.1.0
---

# Reprise Session Close (v2)

Recommended at the end of a Reprise MCP session: one `platform_summary_report` call and one `platform_friction_report` call per distinct friction point. End-of-session is exactly when models forget to call cleanup tools — that's why this is a triggered skill rather than instructions buried in tool descriptions.

## The two tools

- **`platform_summary_report`** — once per session.
- **`platform_friction_report`** — once per distinct friction point.

Both are cross-product (`platform_`-prefixed) and live on every scoped endpoint (`/v2/mcp/`, `/v2/mcp/tour/`, `/v2/mcp/injection/`, `/v2/mcp/clone/`).

## What to call

- **`platform_summary_report` — once per session.** Plain tool (no elicitation). Markdown is fine in `outcome_summary` — include "what was configured" and "what's left open." Round `time_spent_seconds` generously. The server returns a `session_id` that later `platform_friction_report` calls can link to.
- **`platform_friction_report` — once per distinct issue.** Worth submitting **even in autonomous sessions** — when the client doesn't advertise elicitation, the tool persists the draft verbatim with `elicitation_confirmed=false`, so it's still useful when no human is there to confirm.

## `platform_friction_report` argument constraints

These are Pydantic-enforced at the wire boundary — they don't always appear in client-rendered tool descriptions, so inline here so your first call isn't rejected:

- `category`: `bug` | `tool_ergonomics` | `platform_gap` | `docs_gap` | `feature_request`
- `severity`: `blocker` | `major` | `minor`
- `reproducibility`: `confirmed` | `suspected` | `one_off` — NOT `always` / `never` / `intermittent` (those are rejected with `must be one of confirmed, suspected, one_off`)
- `product`: `clone` | `tour` | `data_injection` | `mcp_general` (wire spellings `replicate`, `html`, `replay` also accepted)
- `summary`: max **300 characters** — longer summaries are rejected by the schema. Put the detail in `details` (up to 5000 chars).

## Rules

- **Never chain multiple friction items.** That's an MCP spec anti-pattern. N issues = N calls.
- **Phrase `summary` as the symptom you observed**, not the fix you'd want. Triage reads `summary` first.

## What counts as a distinct issue

A specific tool returning a wrong / misleading error; a documented workflow that didn't work as documented; a capability gap that cost you time; a runtime bug that blocked progress — each is one item. Multiple symptoms of one root cause = one item (mention the symptoms in `details`). Two unrelated problems = two items.

## When to call

End-of-session signals from the user: "we're good", "that's it", "thanks", "we can wrap", topic switch, or an explicit "log this and move on." In autonomous sessions: when the agent has completed the task and is composing its final summary.

If you've already called `platform_summary_report` for this session, don't call again. If you forgot a `platform_friction_report` for a known issue, file it now — late is better than missing.
