# Skip Reasons → Fixes

The `list_lookup(kind="SkipReason")` endpoint returns these codes. When a campaign has high skip rates, the dominant reason points to the fix.

## `Age` / `PostOutdated`
The post is older than the campaign's max-post-age threshold (`Age` = caught at ingestion; `PostOutdated` = aged out between discovery and commenting).

**Fix:**
- If many recent-enough posts are being skipped → keywords too narrow, targets aren't posting fresh content. Broaden keywords or add active targets via `modify_campaign_targets`.
- If acceptable → no action; the filter is doing its job.
- Want a longer window? Inspect `get_campaign_assistant_config(campaign_id).post_filter.skip_posts_older_than_days` and raise it via `update_campaign_assistant(campaign_id, post_filter_patch={skip_posts_older_than_days: <new>})`.

## `ContentFilter` / `InitialContentCheck` / `InitialContentCheckViAssistant`
The content filter rejected the activity. It matches `content_filter.instruction` against BOTH the **post text** and the author's **LinkedIn Headline** in one call — so an author/role rule (e.g. "skip students/recruiters") that fails on the Headline also lands here, not under ProfileFilter. `InitialContentCheck*` variants are the pre-LLM gate; `ContentFilter` is the AI-applied gate.

**Fix:**
- High rate (>40%) → keywords are pulling in irrelevant posts, OR the instruction is too strict on the post text or the author Headline. Tighten keyword phrasing / add negative terms via `modify_campaign_targets`.
- Could also mean the content filter is too aggressive. Inspect the current rule via `get_campaign_assistant_config(campaign_id).content_filter.instruction` and propose a softer version through `update_campaign_assistant(campaign_id, content_filter_patch={instruction: '…'})`. If the over-rejection is author-based, loosen the author/role clause specifically.

## `ProfileFilter` / `ProfileCheckViaAssistant`
This single bucket conflates several distinct causes — the code does not split them out:
- **geo mismatch** — the author's LinkedIn geo is outside `geo_whitelist` / inside `geo_blacklist`, OR
- **profile parse failure / timeout** — the author profile couldn't be fetched in time (profile filtering forces a per-post profile parse), OR
- **wrong author type** — `post_source_filtering_type` excluded a person/company post.

Note: the profile filter is **geo only**. Author/role/Headline rejections come through **ContentFilter**, not here. `filter_open_to_work` / `filter_hiring` are **not enforced by the live pipeline** — do not attribute ProfileFilter skips to those.

**Fix:**
- High rate (>40%) with the profile filter enabled → the geo scope is too tight, or profiles aren't parsing in time. Read `get_campaign_assistant_config(campaign_id).profile_filter`. Widen/clear `geo_whitelist`/`geo_blacklist`, or disable the profile filter entirely (`profile_filter_patch={is_enabled:false}`) if geo scoping isn't essential — that also removes the per-post parse cost/latency.
- Check **Post Author Type** on the campaign: `list_campaigns(id=...).post_source_filtering_type`. If it's `"exclude_person"` / `"exclude_company"` and that's over-narrowing, relax via `update_campaign(campaign_id, patch={ post_source_filtering_type: "none" })`.
- ByProfile campaign with this skip → targets failing parse or out of geo. Use `get_per_target_performance` to find the blocked `target_id`s and propose removal via `modify_campaign_targets.ids_to_remove`.

## `OutOfLimit`
Hit a daily/weekly/monthly per-profile or per-campaign limit.

**Fix:**
- If `post_limit_per_day` is being hit consistently → success, possibly raise via `update_campaign(campaign_id, patch={post_limit_per_day: <new>})` if budget allows.
- If `post_limit_per_profile_per_*` is the binding constraint → user is over-commenting on individual people. Usually correct as protection; rarely needs raising. Diversify via `modify_campaign_targets` instead.

## Other skip codes you may see

- `ContactIgnored` — author is on the user's manual ignore list. Expected; no action.
- `ProfileStat` / `CampaignStat` / `AccountStat` — internal volume guards. Usually transient; flag only if they dominate over multiple days.
- `User` (manual) — the user themselves skipped the activity from the dashboard. Expected.

## Status codes that aren't skip reasons but look similar

These come from `list_lookup(kind="CampaignStatus")`, not skip reasons — don't confuse them in the audit:

- `WaitingForProfile` — searching for the next target, normal transient state
- `WaitingForComment` — generating an AI reply, normal transient state
- `WaitingForNextIteration` — round complete, waiting for `minimal_round_interval` to elapse
- `Failed` — actual error; if persistent, flag for support

A campaign sitting in `Failed` for hours is a real problem; the others usually aren't.
