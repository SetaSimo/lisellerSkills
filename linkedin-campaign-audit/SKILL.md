---
name: linkedin-campaign-audit
description: Performs a weekly performance audit of the user's LinkedIn outreach campaigns (AI commenting on posts for personal/product promotion). Use this skill whenever the user asks for a "weekly review", "campaign audit", "performance check", "how are my campaigns doing", "what's wrong with my campaign", or any retrospective on outreach performance — even if they don't explicitly say "audit". Also use proactively at the start of a new week if the user mentions reviewing or planning around campaigns. Produces a structured report covering throughput vs limits, skip rate breakdown, budget burn, target relevance, and concrete action items.
---

# LinkedIn Campaign Audit (Weekly Performance Review)

This skill produces a weekly performance audit of LinkedIn AI-commenting campaigns. The product searches LinkedIn posts and generates AI replies on them so the user can promote themselves or their product through visibility in comment sections.

> **Output rule (everything the user reads must be user-friendly):** in every line and summary, use
> plain prose only — **never** show UUIDs/GUIDs, raw booleans (say "running"/"paused", "yes"/"no"),
> snake_case field names, MCP tool names, or lookup codes. Translate each to plain English ("23 of 30
> daily comments used", "73% skipped as off-topic", "Pro", "I'll raise the limit"). This covers
> anything pulled from `references/` too — translate before showing. Full do/don't list with examples:
> `references/output-rules.md`.

## When to use

- User asks for a weekly review, audit, retrospective, or performance check
- User says something is off ("my campaign isn't working", "why so few comments today")
- Start-of-week planning conversations
- Before making major changes — audit first to establish baseline

## Recurring audits

If the user wants this audit to repeat on a schedule (daily, weekly, every N days, a specific day-of-week, etc.), do **NOT** try to persist the schedule on the backend — the server does not store audit schedules and exposes no tool for that. Instead, hand off to Claude Code's `/schedule` skill (or `/loop` for self-paced loops) with this same audit prompt as the recurring command. After scheduling, confirm in plain language what was set ("I'll run this audit every Monday at 9am"). If the user later asks to change the cadence or cancel it, that's also Claude Code's `/schedule` skill — not a backend operation.

## Audit checklist (run in order)

For each step, call the listed MCP tool, then evaluate against the criteria. **Do not skip steps** — a partial audit hides root causes.

Default audit window: **last 7 days inclusive** (today minus 6, through today). All date-range tools take `from_date` and `to_date` as `YYYY-MM-DD` strings.

### Step 1 — Pull the campaign list with details

Call `list_campaigns` with `areDetailsRequired=true` (required to populate the `schedule[]` array — without the flag it comes back empty even for campaigns that have schedule windows configured). This returns each campaign with:
- campaign_type (ByProfile or ByKeyword)
- today's posts/comments/likes counts
- round progress
- ai_assistant_method (Off / Common / Pro — Pro is the expected default; Custom must never be set)
- minimal_round_interval or campaign schedule
- limits (post_limit_per_day, per_profile_per_day/week/month)

Note which campaigns are `is_enabled=false` — those are paused and excluded from active analysis (but flag them in the report).

### Step 2 — Pull account context

- `get_user_stats` → current USD balance, monthly USD, and per-action prices (search, content_filter, profile_filter, comment, premium_comment)
- `list_lookup(kind="SkipReason")` → reference table for interpreting `skip_reason_id` / `skip_reason_code` returned by the breakdown endpoint

### Step 3 — Pull weekly data (tiered — one campaign per pass)

> **Budget note (so a full run fits even on a constrained plan):** audit **one campaign per pass**.
> If the user named a campaign, use it. Otherwise pick the **single most-active enabled campaign**
> (most comments today, from the Step 1 snapshot), audit it, then offer to repeat the pass for the
> others. **Do not** fan out the deep calls for every enabled campaign at once. The full report can
> be produced from Tier 0 + Tier 1; Tier 2 only enriches specific findings. A user on a paid plan can
> ask for all campaigns in one go.

For the campaign being audited, pull data in tiers (each takes `campaign_id`, `from_date`, `to_date`):

**Tier 1 — core diagnostics (always; the two-call minimum that drives the TL;DR and most action items):**
- `get_campaign_history` → daily `posts_found / activities_passed / comments_made / likes_made` series. Detects trends (collapsing throughput, dropping activities_passed → posts_found ratio) the one-day `list_campaigns` snapshot hides. **Skip counts are NOT in this stream** — that's `get_skip_breakdown`.
- `get_skip_breakdown` → counts per `skip_reason_id` with `percentage` — **why** activities were skipped (Age / ContentFilter / ProfileFilter / PostOutdated / OutOfLimit). The single most important diagnostic.

**Tier 2 — on-demand only when a Tier-1 signal calls for it (don't pull these by default):**
- `get_campaign_assistant_config` → **only when ContentFilter/ProfileFilter dominates skips** — shows what the filter is currently set to, to explain the dominance.
- `get_per_target_performance` + `get_campaign_targets` (page 0, size 50; paginate only if `total_count > 50`) → **only when throughput is low or the user asks about dead keywords**. Cross-reference the two to name dead targets (0 comments over the window) and suggest removals via `modify_campaign_targets.ids_to_remove`.
- `get_engagement_feedback` → summary + per-comment array sorted by engagement (pull top-3 for the report). Fetch once **if the account is stats-connected**. **If not connected, likes/replies come back zero** — don't report "no engagement"; tell the user in plain language to connect the account for stats on **https://app.liseller.com** under **Analytics** (https://app.liseller.com/pages/analytics) and **Inbox** (https://app.liseller.com/pages/communications), and skip this call.

### Step 4 — Evaluate each enabled campaign against the checklist

See `references/audit-checklist.md` for the full per-campaign scorecard. The five core dimensions are:

1. **Throughput health** — is the campaign hitting its `post_limit_per_day` or falling short? Falling short by >40% on multiple days = under-targeted (not enough relevant posts found) or over-filtered. Use `get_campaign_history` for the day-by-day picture, not just the `list_campaigns` snapshot.
2. **Skip rate breakdown** — from `get_skip_breakdown`: which `skip_reason_code` dominates? Each cause maps to a specific fix (see `references/skip-reasons.md`).
3. **Budget burn** — at current daily comment count × per-action price, how many days does the balance last? Flag if <14 days runway or if monthly_usd is being burned faster than 1/30 per day.
4. **AI method fit** — **Pro is the expected method for every campaign.** `Off` campaigns generate likes only, no visibility lift. `Common` lacks the research-backed quality of Pro. Any campaign on `Common` or `Off` → raise a "move to Pro" action item. `Custom` must never be set (UI-only). If engagement data was pulled (Tier 2), cross-reference `engagement_feedback.summary.avg_likes_per_comment` to quantify the gap — otherwise the recommendation for a non-Pro campaign stands regardless: move it to Pro.
5. **Schedule sanity** — if `is_using_schedule=true`, are the windows wide enough to hit the day limit? Narrow windows + high `pause_between_posts_in_minutes` = arithmetic impossibility.

### Step 5 — Cross-campaign issues

- Multiple campaigns sharing the same `li_account_id` and both hitting `OutOfLimit` skips? → LinkedIn-side throttling, lower per-campaign limits.
- Global `pause_between_posts_in_minutes` × (sum of daily limits) > minutes-in-active-windows? → mathematically can't hit targets.

### Step 6 — Write the report

Use the template in `references/report-template.md`. Structure:

- **TL;DR** — 3 bullets max: biggest win, biggest problem, top action
- **Per-campaign table** — name, status, today's comments/limit, dominant skip reason, health verdict (✅ healthy / ⚠️ needs attention / 🔴 broken)
- **Account-level findings** — budget runway, global throttling issues
- **Top engagement** — top 3 AI comments by likes+replies (post URL, content snippet, counts)
- **Action items** — concrete and prioritized: which campaign, which tool call, what change. **Do not execute these calls yourself** — propose them, get the user's go-ahead, then apply.

Keep the language direct and quantified. "Campaign X hit 12/50 daily limit, 73% skipped on ContentFilter" beats "Campaign X is underperforming."

## Important: do not mutate state during an audit

This skill is **read-only**. Never call `update_campaign`, `update_campaign_assistant`, `modify_campaign_targets`, or `delete_campaign` while running the audit. Propose those calls as action items in the report; let the user approve, or pivot to the `linkedin-growth-recommendations` skill to apply them.

## Missing endpoints (still blind to these)

Most of the audit is fully covered now. The only remaining gap is keyword expansion preview:

- **`validate_keywords(keywords[])`** — server-side dry-run of new keywords against LinkedIn search to preview volume and sample posts. Not in scope for v1. The audit doesn't actually need this — keyword expansion belongs in the growth skill — but mention it if the user asks "can I test a keyword before adding it?".

## Detailed references

- `references/output-rules.md` — full do/don't list for user-facing output
- `references/audit-checklist.md` — full per-campaign scorecard with thresholds
- `references/skip-reasons.md` — what each skip reason means and how to fix it
- `references/report-template.md` — exact report structure with placeholders
