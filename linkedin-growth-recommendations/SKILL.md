---
name: linkedin-growth-recommendations
description: Analyzes a mature LinkedIn outreach campaign (AI commenting on posts) and produces specific recommendations for growth — expanding keywords, adjusting filters, raising limits, upgrading the AI method, or pruning dead targets. Use this skill whenever the user asks "how do I grow my campaign", "what should I change", "campaign is plateauing", "I want more reach", "should I add more keywords", "comments aren't being seen", or any optimization/scaling question on an existing campaign. Also use after an audit when the user wants to act on findings. Distinguishes growth (scaling up what works) from setup (starting fresh) — don't run this on a campaign younger than 2 weeks or with no performance history.
---

# LinkedIn Growth Recommendations

This skill is for *scaling up* a working campaign, not fixing a broken one and not setting up a new one. The product comments on LinkedIn posts via AI; "growth" here means more visibility on more relevant posts, not just more comments.

> **Output rule (everything the user reads must be user-friendly):** in every line and summary, use
> plain prose only — **never** show UUIDs/GUIDs, raw booleans (say "running"/"paused", "yes"/"no"),
> snake_case field names, MCP tool names, or lookup codes. Translate each to plain English ("the
> daily comment limit", "skipped as off-topic", "Pro", "I'll raise the limit"). This covers anything
> pulled from `references/` too (the **Template** / `Tool call:` lines in `recommendation-patterns.md`
> are agent-only) — translate before showing. Full do/don't list: `references/output-rules.md`.

## When to use

- User has an existing campaign with 2+ weeks of history and wants more reach
- User says "plateau", "flat", "stuck", "want to scale", "expand", "grow"
- Right after an audit, when the user wants to act on findings
- User mentions adding keywords/profiles, raising limits, or changing filters

## When NOT to use

- New campaign (<2 weeks old) — run `linkedin-campaign-setup` instead, or wait
- Campaign is broken (0 throughput, high failure rate) — run `linkedin-campaign-audit` first to fix the foundation
- User wants entirely new positioning — that's a new campaign, not growth

## Workflow

Default analysis window: **last 14 days inclusive** (today minus 13, through today). All date-range tools take `from_date` and `to_date` as `YYYY-MM-DD` strings.

### Step 1 — Confirm preconditions and pull data (tiered)

> **Budget note (so a full run fits even on a constrained plan):** classify the bottleneck from the
> minimal Tier-1 set first, then pull deeper Tier-2 data **only for the bottleneck you actually
> found** — don't fan out all eight calls up front.

**Tier 1 — preconditions + bottleneck classification (the Step-2 decision tree needs only these):**
- `list_campaigns(id=<campaign_id>, areDetailsRequired=true)` — verify `is_enabled=true`, status not stuck on `Failed`/`WaitingForProfile`, today's comments >50% of `post_limit_per_day`. `areDetailsRequired=true` also populates `schedule[]` (otherwise empty even for scheduled campaigns).
- `get_campaign_history(<campaign_id>, from_date, to_date)` — daily series for trend analysis.
- `get_skip_breakdown(<campaign_id>, from_date, to_date)` — primary diagnostic signal.
- `get_user_stats()` — balance + plan + per-action prices, for cost-impact estimates and whether paid-tier moves (Pro AI) are realistic.

If preconditions fail (not enabled, mostly idle, stuck status) → stop, recommend running `linkedin-campaign-audit` first. Growth on a broken campaign just wastes more budget.

**Tier 2 — fetch only for the bottleneck identified in Step 2:**
- Target-bound / concentration-bound → `get_per_target_performance(<campaign_id>, from_date, to_date)` + `get_campaign_targets(<campaign_id>, page_number=0, page_size=50)` (paginate if `total_count > 50`). Cross-reference to identify dead targets by `target_id`.
- Filter-bound → `get_campaign_assistant_config(<campaign_id>)` — current filter config you'd be patching.
- Quality-bound → `get_engagement_feedback(<campaign_id>, from_date, to_date)` — likes/replies; pull top-3 as a quality reference.

### Step 2 — Identify the growth bottleneck

There are five distinct bottlenecks, each with a different fix. See `references/bottleneck-diagnosis.md` for full diagnosis.

Quick decision tree:

| Symptom | Bottleneck | Fix lever |
|---|---|---|
| Hitting daily limit, budget allows more | **Limit-bound** | Raise `post_limit_per_day` |
| Far below daily limit, high `ContentFilter` skips | **Filter-bound** | Tighten or expand keywords; revisit filter settings |
| Far below daily limit, low skip rate | **Target-bound** | Expand keyword/profile list |
| At limit but comments get no engagement | **Quality-bound** | Upgrade AI method (Common → Pro) |
| At limit but only a few profiles get repeat coverage | **Concentration-bound** | Diversify targets, raise per-profile limits |

### Step 3 — Generate specific recommendations

For each bottleneck identified, produce 2–4 concrete actions. Every action must be:
- **Specific** — name the parameter and value, not "increase limits"
- **Justified** — tied back to a number from the data
- **Estimable** — give expected daily cost impact from price data

See `references/recommendation-patterns.md` for canonical patterns and worked examples.

### Step 4 — Prioritize

Order recommendations by expected impact ÷ effort:

1. **High-impact, low-effort:** raise limits when budget allows, prune dead keywords
2. **High-impact, medium-effort:** expand targets, upgrade AI method
3. **Medium-impact, low-effort:** schedule tweaks, round interval changes
4. **Investigate further:** anything that requires a missing endpoint

Present no more than **5 recommendations** at a time. More than 5 = the user won't act on any of them.

### Step 5 — Recommend a measurement plan

Tell the user: "Apply 1–2 of these changes, then run the audit again in 7 days. Don't change everything at once — you won't know what worked."

If the user wants to apply changes now, use the appropriate write tool:
- Parameter changes (limits, AI method, schedule, account binding) → `update_campaign(campaign_id, patch={…})`.
- Target additions/removals → `modify_campaign_targets(campaign_id, items_to_add=[…], ids_to_remove=[<target_id>, …])`. `target_id` values come from `get_per_target_performance` / `get_campaign_targets`.
- Filter changes → `update_campaign_assistant(campaign_id, content_filter_patch?, profile_filter_patch?, post_filter_patch?)`. At least one patch must be non-null. Triggers re-evaluation of pending activities on the next round.

**Always present the diff before applying** and get explicit approval.

## Missing endpoints (still blind to these)

Most growth analysis is now end-to-end. Two gaps remain:

1. **`validate_keywords(keywords[])`** — server-side preview of new keywords against LinkedIn search (volume, sample posts) before adding them via `modify_campaign_targets`. Not in scope for v1. The fallback is the audit-grow-audit loop: add 5–10 keywords, wait one round/day, run `linkedin-campaign-audit` to see what stuck.
2. **`suggest_related_keywords(seed_keywords[])`** — server-side semantic expansion from working keywords. Not implemented; the agent must brainstorm from general knowledge using `references/keyword-expansion.md`. For niche domains, ask the user for vocabulary instead of inventing it.

## Detailed references

- `references/output-rules.md` — full do/don't list for user-facing output
- `references/bottleneck-diagnosis.md` — full decision tree for identifying the limiting factor
- `references/recommendation-patterns.md` — canonical recommendation patterns with worked examples
- `references/keyword-expansion.md` — how to brainstorm new keywords when expansion is the right move
