# Bottleneck Diagnosis

A growing campaign has exactly one of five bottlenecks at any time. Identify which, then fix that one — don't shotgun changes.

## How to identify

Walk through the questions in order. Stop at the first "yes".

### 1. Limit-bound?

**Test:** is `today_comments >= post_limit_per_day × 0.85` for most days in the recent window?

If yes → **Limit-bound**. The campaign is hitting its ceiling. The fix is to raise the ceiling (if budget allows) or upgrade quality at the same volume.

### 2. Filter-bound?

**Test:** from `get_skip_breakdown` — is throughput far below the daily limit AND the dominant `skip_reason_code` is `content_filter` (or `initial_content_check*`) or `profile_filter` (or `profile_check_via_assistant`)?

If yes → **Filter-bound**. The campaign is finding posts but rejecting them. Two sub-cases:
- **ContentFilter dominant** → keywords are pulling in topically-irrelevant posts, OR the content rule is too strict. Note the `content_filter.instruction` is matched against both the post text AND the author Headline, so an over-strict author/role clause also shows up here. Fix keywords via `modify_campaign_targets`, OR loosen `content_filter.instruction` via `update_campaign_assistant`. Use `get_campaign_assistant_config(campaign_id).content_filter` to see the current rule first.
- **ProfileFilter dominant** → the profile filter is **geo only**: author geo is out of scope, or profiles aren't parsing in time. Check `get_campaign_assistant_config(campaign_id).profile_filter`. Widen/clear geo, or disable the profile filter (`profile_filter_patch={is_enabled:false}`) if geo scoping isn't needed. For ByProfile, the target list may have drifted — use `get_per_target_performance` to find which `target_id`s have high `skip_breakdown.profile_filter` and propose removal. (Author/role rejections are **ContentFilter**, not here.)

### 3. Target-bound?

**Test:** is throughput far below the daily limit AND skip rate is low?

If yes → **Target-bound**. The campaign isn't finding enough posts to begin with. The keyword set is too narrow or the profile list isn't posting enough. Expand.

### 4. Quality-bound?

**Test:** from `get_engagement_feedback` — the campaign is hitting its limit, but `summary.avg_likes_per_comment < 0.5` and `summary.engagement_rate < 0.2`. Pull `comments[]` and look at the top 3 by engagement vs the median — if even the best comments are getting <2 reactions, that's quality-bound.

If yes → **Quality-bound**. Volume is fine, but visibility per comment is low. The AI method or the voice is wrong. If the campaign is on Common, move it to Pro (Pro is the baseline method); otherwise revisit the voice configuration via `update_campaign(campaign_id, patch={ai_assistant_method_id: …})` (get the new id from `list_lookup(kind="AiMethod")`). Never select Custom.

### 5. Concentration-bound?

**Test:** hitting limits but the same handful of profiles dominate the comment log. `post_limit_per_profile_per_week` is binding before `post_limit_per_day` is.

If yes → **Concentration-bound**. A few prolific posters are absorbing all the capacity. Either raise per-profile limits (if those profiles really are the priority) or expand targets to dilute.

## Multiple bottlenecks?

Possible but rare. If two seem to apply, **fix the upstream one first**:

- Filter-bound dominates Target-bound (fixing filters might raise throughput enough that you don't need new targets)
- Target-bound dominates Limit-bound (no point raising a limit you can't hit)
- Concentration-bound is usually downstream of Target-bound

## What if the data is ambiguous?

If two bottlenecks look equally plausible (e.g. throughput is mid-range and skip_breakdown is roughly balanced between ContentFilter and a low total skip rate), **say so**. "Symptoms are between Filter-bound and Target-bound — recommend tightening the worst keyword first, then re-audit in a week to see if throughput rises." beats a confident guess that turns out wrong.
