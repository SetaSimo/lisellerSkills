---
name: linkedin-campaign-setup
description: Guides the user through setting up a new LinkedIn outreach campaign from scratch (AI commenting on LinkedIn posts for personal or product promotion). Use this skill whenever the user wants to "create a campaign", "set up outreach", "start commenting on LinkedIn", "launch a new campaign", or describes a new audience/topic they want to engage with — even if they don't say "campaign". Also use when the user is migrating from another tool or describes a positioning goal that implies needing a campaign. Walks through campaign type, keyword/profile target design (including finding ICP profiles with search_profiles), filters, AI method, persona, schedule, limits + ramp-up, and a dry-run before enabling. Built to run several campaigns back-to-back in one chat without losing track. Prevents the common mistake of enabling a campaign before validating targets.
---

# LinkedIn Campaign Setup (From Scratch)

This skill helps create a new LinkedIn AI-commenting campaign correctly the first time. The product searches LinkedIn posts and generates AI replies on them; getting the setup right means the right posts get found and the right voice replies.

The backend gates `set_campaign_running(enabled=true)` on `is_setup_completed=true`, non-empty `ai_brief`, at least one target, and configured schedule windows (if `is_using_schedule=true`). **The campaign cannot run until you have explicitly marked it complete** — and that is done **only** through `complete_campaign_setup` (Milestone 6). `update_campaign` no longer sets `is_setup_completed`. `is_enabled` is not exposed via `update_campaign` either — start/stop only via `set_campaign_running`.

> **Output rule (everything the user reads must be user-friendly):** in every message and summary, use
> plain prose only — **never** show UUIDs/GUIDs, raw booleans (say "running"/"paused", "yes"/"no"),
> snake_case field names, MCP tool names, lookup codes, or internal entity names. Translate each to plain
> English ("the daily comment limit", "I'll create the campaign and add the keywords", "Pro"). This covers
> anything pulled from `references/` too — translate before showing. Full do/don't list with examples:
> `references/output-rules.md`.

## When to use

- User wants to create a new campaign
- User describes a goal that implies a new campaign ("I want to be seen by founders in fintech")
- User is migrating from a competitor tool and rebuilding their setup
- User has an existing campaign but the goal has shifted enough that a fresh setup is cleaner than editing

## How to run this skill (progressive disclosure)

The setup is split into **milestones**. Each milestone has its own reference file with the exact tool
calls, defaults, and rules for that phase. **Open one milestone file at a time, do that phase, then drop
it from focus and move to the next** — don't load the whole flow at once. This is what lets you set up
several campaigns in a single conversation without running out of context. The deeper topic references
(persona, targets, filters, limits) are linked from the milestone that needs them; read those on demand
too.

### Milestone map (run in order)

- **M0 — Setup context + goal** → `references/milestone-0-context-and-goal.md`
- **M1 — Campaign type + supporting data + targets** → `references/milestone-1-type-and-targets.md` (ByKeyword: `references/target-design.md`; ByProfile discovery: `references/profile-search.md`)
- **M2 — Create campaign (disabled) + filters** → `references/milestone-2-create-and-filters.md` (`references/content-filter-examples.md`)
- **M3 — AI method + persona + comment type** → `references/milestone-3-method-and-persona.md` (`references/persona-design.md`, `references/ui-default-templates.md`)
- **M4 — Cadence, limits + ramp-up, name, auto-toggles** → `references/milestone-4-cadence-limits-name-toggles.md` (`references/limits-and-methods.md`)
- **M5 — Dry-run review (+ cost & runway)** → `references/milestone-5-review.md` (`references/setup-checklist.md`)
- **M6 — Complete setup, launch, follow-up** → `references/milestone-6-complete-and-launch.md`

## Running several campaigns in one chat (read this up front)

You can set up multiple campaigns in the same conversation. Tell the user up front: once one campaign is
launched, they can start another right away — just describe the next audience or goal and you'll run the
setup again from M0. When building multiple campaigns, give **each one its own focused ~30 keywords** for
its theme — don't split a single 30-keyword set across them. Heads-up: the user's plan caps how many
campaigns they can have, so creating one may be rejected with a "campaign limit reached" error — if that
happens, explain it in plain language and offer to pause/delete an unused campaign or upgrade the plan
rather than silently failing.

> **Session ledger (keep this current the whole chat).** Maintain a short running list of the campaigns
> you create this session: each one's plain-language name and whether it is **launched** or **awaiting
> launch**. Don't rely on scrolling back through full context — keep the ledger compact and re-read it
> when you finish a campaign or switch tasks. It is how you stay oriented across 5 campaigns.

> **Always offer to launch — even mid-next-campaign.** When a campaign reaches setup-complete you MUST
> offer to start it. If the user has already begun describing the next campaign before launching the
> previous one, do not silently move on — explicitly ask whether to switch on the just-finished campaign
> first, then record the answer in the ledger. Never end the conversation leaving a completed campaign
> un-launched unless the user explicitly chose to leave it paused. Full handling in
> `references/milestone-6-complete-and-launch.md`.

## Detailed references

- `references/milestone-0-context-and-goal.md` … `references/milestone-6-complete-and-launch.md` — the seven milestone driver files (open one at a time)
- `references/profile-search.md` — finding ICP profiles for ByProfile campaigns with `search_profiles` (full filter vocabulary + workflow)
- `references/target-design.md` — how to write good keywords (ICP-driven) and pick good profile targets, with worked examples
- `references/limits-and-methods.md` — full guidance on AI methods, limits, ramp-up, intervals, and schedule semantics
- `references/persona-design.md` — how to fill the persona / product / comment-type fields, OpenerLines examples
- `references/persona-examples.md` — compressed real-world persona fills (product, USP, persona, ai_brief, tone) to model concreteness
- `references/ui-default-templates.md` — verbatim UI default templates with `{{...}}` placeholders for every persona field. Use these as the baseline rather than inventing new shapes.
- `references/content-filter-examples.md` — real-world content-filter instruction patterns
- `references/setup-checklist.md` — final pre-launch checklist
- `references/output-rules.md` — what never to show the user (translate all internals to plain English)
