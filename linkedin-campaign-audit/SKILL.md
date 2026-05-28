---
name: linkedin-campaign-audit
description: Performs a weekly performance audit of the user's LinkedIn outreach campaigns (AI commenting on posts for personal/product promotion). Use this skill whenever the user asks for a "weekly review", "campaign audit", "performance check", "how are my campaigns doing", "what's wrong with my campaign", or any retrospective on outreach performance — even if they don't explicitly say "audit". Also use proactively at the start of a new week if the user mentions reviewing or planning around campaigns. Produces a structured report covering throughput vs limits, skip rate breakdown, budget burn, target relevance, and concrete action items.
---

# LinkedIn Campaign Audit (Weekly Performance Review)

This skill produces a weekly performance audit of LinkedIn AI-commenting campaigns. The product searches LinkedIn posts and generates AI replies on them so the user can promote themselves or their product through visibility in comment sections.

> **Output rule (everything the user sees must be human-friendly):** every line you render — TL;DR, tables, per-campaign findings, top-engagement entries, action items, follow-ups — uses plain prose only. **Never** show: (a) UUIDs / GUIDs (campaign ids, target ids, account ids, status ids — translate by name or omit); (b) raw booleans `true` / `false` — say "running" / "paused", "enabled" / "disabled", "yes" / "no"; (c) raw API field names or snake_case keys (`is_enabled`, `skip_reason_code`, `ai_assistant_method_id`, `post_limit_per_day`, `content_filter`, etc.) — translate every one (e.g. "23 of 30 daily comments used", "73% skipped as off-topic", "Pro", "the daily comment limit"); (d) MCP tool names (`mcp__claude_ai_Liseller__list_campaigns`, `mcp__claude_ai_Liseller__get_engagement_feedback`, or any other `mcp__…` slug, or even the bare `list_campaigns` / `update_campaign` strings) — describe the action in English ("I'll raise the limit", not "call `update_campaign`"); (e) lookup codes (`premium`, `ContentFilter`, `WaitingForProfile`) — say "Pro", "off-topic skips", "waiting on profile data". This rule applies to **every line you render for the user**, including anything pulled from `references/report-template.md` — translate before showing. Raw field names and tool names exist only for tool calls; they must never appear in what the user reads.

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

### Step 3 — Pull weekly history per enabled campaign (parallel)

For every `is_enabled=true` campaign, fan out these calls in parallel (each takes `campaign_id`, `from_date`, `to_date`):

- `get_campaign_history` → daily series of `posts_found / activities_passed / comments_made / likes_made` (sourced from ClickHouse analytics events). Use this to detect trends (collapsing throughput, dropping activities_passed → posts_found ratio) instead of relying on the one-day snapshot from `list_campaigns`. **Skip counts per day are intentionally NOT in this stream** — call `get_skip_breakdown` for the same window to see why posts dropped out.
- `get_skip_breakdown` → counts per `skip_reason_id` with `percentage`. This tells you **why** activities were skipped (Age / ContentFilter / ProfileFilter / PostOutdated / OutOfLimit, etc.) — the most important diagnostic signal.
- `get_engagement_feedback` → summary (total comments, likes/replies received, avg likes per comment, engagement rate) plus a per-comment array sorted by engagement. This is the truest success metric for a visibility product. Pull the top-3 comments for the report. **If the account is not connected for statistics, likes/replies come back as zero** — don't report "no engagement"; instead tell the user, in plain language, to connect the account for stats on **https://app.liseller.com** under **Analytics** (https://app.liseller.com/pages/analytics) and **Inbox** (https://app.liseller.com/pages/communications). Check the account's stats-connection status first.
- `get_per_target_performance` → per-target counters (`posts_found / comments_made / likes_made / skipped_count` + per-target `skip_breakdown`). Use this to spot dead keywords/profiles (0 comments over the full window).
- `get_campaign_assistant_config` → current `content_filter` / `profile_filter` / `post_filter` config. Needed to interpret skip-reason dominance ("ContentFilter is 70% of skips → here is what the filter is currently set to").
- `get_campaign_targets` (page 0, size 50, paginate if `total_count > 50`) → the actual keywords/URLs driving the campaign. Cross-reference with `get_per_target_performance` to name dead targets and suggest removals via `modify_campaign_targets.ids_to_remove`.

### Step 4 — Evaluate each enabled campaign against the checklist

See `references/audit-checklist.md` for the full per-campaign scorecard. The five core dimensions are:

1. **Throughput health** — is the campaign hitting its `post_limit_per_day` or falling short? Falling short by >40% on multiple days = under-targeted (not enough relevant posts found) or over-filtered. Use `get_campaign_history` for the day-by-day picture, not just the `list_campaigns` snapshot.
2. **Skip rate breakdown** — from `get_skip_breakdown`: which `skip_reason_code` dominates? Each cause maps to a specific fix (see `references/skip-reasons.md`).
3. **Budget burn** — at current daily comment count × per-action price, how many days does the balance last? Flag if <14 days runway or if monthly_usd is being burned faster than 1/30 per day.
4. **AI method fit** — **Pro is the expected method for every campaign.** `Off` campaigns generate likes only, no visibility lift. `Common` lacks the research-backed quality of Pro. Any campaign on `Common` or `Off` → raise a "move to Pro" action item. `Custom` must never be set (UI-only). Cross-reference `engagement_feedback.summary.avg_likes_per_comment` to quantify the gap, but the recommendation for a non-Pro campaign is always: move it to Pro.
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

- `references/audit-checklist.md` — full per-campaign scorecard with thresholds
- `references/skip-reasons.md` — what each skip reason means and how to fix it
- `references/report-template.md` — exact report structure with placeholders
