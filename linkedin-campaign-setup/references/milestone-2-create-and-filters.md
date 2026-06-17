# Milestone 2 — Create the campaign (disabled) + filters

Read this after Milestone 1. It covers the original Steps 5–6. When you finish it, return to the
milestone map in `SKILL.md` and open the Milestone 3 file.

## Step 5 — Create the campaign (DISABLED) and add targets

`create_campaign(patch={ campaign_type_id, li_account_id, li_organization_id? })`. **Pass `campaign_type_id`** from `list_lookup(kind: "CampaignType")` (Milestone 1) — defaults to ByKeyword if omitted, but be explicit. **Do not pass `name` yet** — it gets a default `"New campaign"` and is renamed in Milestone 4 (closer to UI flow). **Do not pass `post_source_filtering_type` yet** — that's the filters step below. The server applies the same defaults the UI uses:
- `post_limit_per_day=10`, per-profile `1/day` / `7/week` / `30/month`
- `skip_posts_older_than_days=90`
- ramp-up enabled with `frequency_in_days=4`, `increment=2`, `max_limit=50`
- assistant method `Off`, content/profile/post filters all off
- `language_whitelist=["English"]` already preset — do not re-send this in the filters step unless the user picked different languages
- `auto_comment_enabled=false`, `auto_like_enabled=false` (do not turn these on yet — see Milestone 4)
- `commenting_mode="neutral"` is the create-time default on the assistant
- `post_source_filtering_type="none"` (Any author)
- `is_setup_completed=false`

Capture the returned campaign id, then `modify_campaign_targets(request={ campaign_id, items_to_add: [...] })` with the keywords or profile URLs from Milestone 1.

## Step 6 — Filters

Apply assistant-level filters via `update_campaign_assistant(campaignId, patch={ content_filter_patch, profile_filter_patch, post_filter_patch })`. Apply campaign-level **Post Author Type** via `update_campaign(campaignId, patch={ post_source_filtering_type })`.

**Post Author Type.** Ask: «companies, individuals, or both?» → `none` (Any, default) / `exclude_person` (Only Companies) / `exclude_company` (Only Users). Send the `code` **string** (not the `id`) from `list_lookup(kind: "PostSourceFilteringType")`.

**Where a rule belongs** (the key distinction — full detail in `references/persona-design.md`):
- **Content filter** (`content_filter_patch.instruction`) — one paid call per post, matched against the post text **and** the author Headline. Put post-substance rules **and** author/role rules ("skip students", "only CTOs") here. No profile parse, no extra cost.
- **Profile filter** (`profile_filter_patch`) — **geo only**; enabling it forces a per-post profile parse (extra latency + cost). Use only for LinkedIn-profile geo scoping.
- `filter_open_to_work` / `filter_hiring` — accepted but **not enforced today**; don't rely on them.

**Content filter — always apply:**
1. **Mandatory opening line:** `I am looking for posts about <broad topic that unites the user's keywords>` (or `…from <broad ICP description>`). Derive the topic from the keyword list.
2. `is_enabled=true` plus the two default objective rules at the end: `Reject posts that contain only a link with no original text.` and `Reject posts that contain only emoji with no substantive text.`
3. **Objective skip rules only** (content shape, literal strings, author role in headline) — **never** subjective ones, and **no "Prioritise" lines** (it's a binary keep/skip gate — to narrow harder, add more `Skip` rules).
4. Language → `language_whitelist=["English"]` (already preset; resend only if the user named other languages). Geo → `profile_filter_patch.geo_whitelist`/`geo_blacklist` + `is_enabled=true`. Max post age → `post_filter_patch.skip_posts_older_than_days=90` (range 0–90).

The opening line + two default rules are not optional. See `references/content-filter-examples.md` for real-world instruction patterns.

> **MCP caps tightened to UI bounds:** `post_limit_per_day` ≤ 200, `post_limit_per_profile_per_day` ≤ 10, `…_per_week` ≤ 50, `…_per_month` ≤ 150. Validator rejects anything above. Don't talk users into "just bump to 1000" — that path is closed.

---

**Milestone 2 done.** Return to the milestone map and open `references/milestone-3-method-and-persona.md`.
