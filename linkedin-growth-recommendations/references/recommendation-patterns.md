# Recommendation Patterns

Canonical recommendation patterns by bottleneck. Each pattern has two parts:

- **Template** — the text you actually render to the user. Plain, human-readable language only — no raw API field names, no MCP call syntax.
- **Tool call:** — agent-only reference for the API the agent will use after the user approves. **Never echo this line to the user.**

## Pattern: "Raise the daily ceiling" (Limit-bound)

**When:** hitting the daily comment limit >85% consistently, runway > 30 days.

**Template:**

> Campaign **{name}** is hitting its daily comment limit of **{X}/day** consistently. Budget runway is **{N} days**, well above the 14-day floor. **Raise the daily comment limit to {1.5×X}** for a week, then audit again. Estimated daily cost rises from **${current}** to **${new}**.

**Tool call:** `update_campaign` with `{post_limit_per_day: 1.5×X}`

**Caveat:** before raising, verify the schedule and `pause_between_posts_in_minutes` make the new limit feasible.

## Pattern: "Prune dead keywords" (Filter-bound, ContentFilter)

**When:** `content_filter` (or `initial_content_check*`) is the dominant `skip_reason_code` on a ByKeyword campaign.

**Template:**

> **{X}% of activities are being skipped by the content filter** — the campaign is finding posts but rejecting them as off-topic. From per-target stats, these {N} keywords account for most of the bad traffic: **{list of profile_url_or_keyword from get_per_target_performance where skip_breakdown.content_filter / skipped_count is highest}**. Recommend removing them.

**Tool call:** `modify_campaign_targets(campaign_id=…, items_to_add=[], ids_to_remove=[<target_id>, …])`. Use `target_id` values straight from `get_per_target_performance` — they match `modify_campaign_targets.ids_to_remove`.

Cross-check before removing: if `get_campaign_assistant_config(campaign_id).content_filter.instruction` is overly strict, soften the rule first via `update_campaign_assistant(content_filter_patch={instruction: '…'})` — that may revive the keyword without removing it.

## Pattern: "Add audience-anchored keywords" (Filter-bound, ProfileFilter on ByKeyword)

**When:** `profile_filter` (or `profile_check_via_assistant`) dominant on a ByKeyword campaign.

**Template:**

> Posts are being found, but **{X}% are by the wrong audience**. Your keywords are topic-anchored but not audience-anchored. Add 3–5 keywords that combine your topic with audience signals — seniority, role, or company stage. Examples for "{user's topic}": **"{anchored example 1}"**, **"{anchored example 2}"**, **"{anchored example 3}"**.

Audience scoping is done through the **content filter**, not the profile filter: the `content_filter.instruction` is matched against the author Headline as well as the post text. So to target/exclude authors by role, add/loosen an author clause in `content_filter.instruction` via `update_campaign_assistant(campaign_id, content_filter_patch={instruction:'…'})` (read it first with `get_campaign_assistant_config(campaign_id).content_filter`). The profile filter is geo-only — widen `geo_whitelist`/`geo_blacklist` or disable it (`profile_filter_patch={is_enabled:false}`) only if geo is the problem. (`filter_open_to_work` / `filter_hiring` are not enforced — changing them does nothing.)

**Tool call:** `modify_campaign_targets(campaign_id=…, items_to_add=['keyword 1', 'keyword 2', …], ids_to_remove=[])` (items_to_add is a flat list of strings).

## Pattern: "Expand keyword set" (Target-bound)

**When:** Far below daily limit, low skip rate, ByKeyword campaign.

**Template:**

> Throughput is **{X}/{Y} per day ({Z}%)** with low skip rate — the keywords are good, there just aren't enough of them. Current set is **{N} keywords**. Add **5–10 new keywords** in adjacent topical territory. See `keyword-expansion.md` for brainstorming patterns.

**Tool call:** `modify_campaign_targets` with `items_to_add` (a flat list of keyword strings).

## Pattern: "Add active posters" (Target-bound, ByProfile)

**When:** Far below daily limit on a ByProfile campaign.

**Template:**

> Throughput is **{X}/{Y} per day** — of the {N} target profiles, **{D} have `posts_found == 0` over the {window} window** (from `get_per_target_performance`). Recommend removing those and adding {D+5} active posters in the same role/company tier. Aim for **10–50 profiles, each posting 2+ times a week**.

**Tool call:** `modify_campaign_targets(campaign_id=…, items_to_add=['https://linkedin.com/in/…', …], ids_to_remove=[<dormant target_id>, …])` (items_to_add is a flat list of URL strings). `target_id` values come from `get_per_target_performance`.

## Pattern: "Move to Pro" (Quality-bound, or any campaign still on Common)

**When:** the campaign is on `Common` (Pro is the baseline method for every campaign), OR engagement on comments is weak.

**Template:**

> Campaign **{name}** is running on **Common**. Pro is the recommended method for every campaign — it produces research-backed, higher-engagement comments. Move it to **Pro**. New per-comment cost: **${new_price}** vs current **${current_price}**. Daily cost impact at the current daily comment limit of {X}: **${delta}**.

**Tool call:** `update_campaign` with `{ai_assistant_method_id: <Pro method id>}` (get the id from `list_lookup(kind="AiMethod")` — the entry whose code is `premium`). Never select Custom.

## Pattern: "Dilute concentration" (Concentration-bound)

**When:** `post_limit_per_profile_per_week` binding before `post_limit_per_day`.

**Template:**

> A few profiles are absorbing all your capacity. The weekly per-profile cap ({P}) is being hit before the daily comment limit ({D}). Two options:
>
> 1. **Add 10+ more targets** to spread capacity (preferred if you want to diversify reach)
> 2. **Raise the weekly per-profile cap to {P+2}** if those profiles really are the priority audience

**Tool call:** either `modify_campaign_targets` (option 1) or `update_campaign` (option 2).

## Pattern: "Widen post age window" (Filter-bound, Age / PostOutdated)

**When:** `age` or `outdated` (PostOutdated) is the dominant `skip_reason_code`.

**Template:**

> **{X}% of activities are being skipped because posts are older than the freshness threshold.** Current maximum post age is **{D} days**. If the audience is in a slower-moving topic (e.g. enterprise, regulated industries), 30 days is often too tight — try **{D + 30} days**.

**Tool call:** `update_campaign_assistant(campaign_id=…, post_filter_patch={skip_posts_older_than_days: {D + 30}})`. Triggers re-evaluation of pending activities on the next round.

## Pattern: "Tighten schedule for human pacing" (cosmetic, low-impact)

**When:** Schedule is overly broad (e.g. 24h/day), or campaign-runs-all-night posting pattern looks botty.

**Template:**

> Posts are landing at all hours, which can look automated. Tighten the schedule to **{audience's working hours in their timezone}** so the activity pattern looks more human.

**Tool call:** `update_campaign` with `{is_using_schedule: true, schedule: [...]}`.

**Caveat:** verify the narrower schedule still mathematically permits the daily limit.

## What NOT to recommend

- **"Use more AI methods at once"** — only one per campaign.
- **"Lower `pause_between_posts_in_minutes`"** without warning the user it's global, not per-campaign.
- **"Select Custom"** — never; Custom is UI-only and cannot be configured via MCP. Pro is the default method for every campaign.
- **"Just add more keywords"** without specifying what kind — leads to dilution, not growth.
