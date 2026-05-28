# Per-Campaign Audit Scorecard

For each `is_enabled=true` campaign, score on these five dimensions and assign a verdict.

## 1. Throughput health

**What to compute:** `today's_comments / post_limit_per_day`

| Ratio | Verdict | Likely cause |
|---|---|---|
| 0.85 — 1.00 | ✅ Healthy | Working as designed |
| 0.50 — 0.85 | ⚠️ Underperforming | Targets producing fewer relevant posts than expected, OR filters rejecting too much |
| < 0.50 | 🔴 Broken | Too few targets, schedule too narrow, OR filters mis-tuned |
| > 1.00 | ❓ Data anomaly | Verify the count — limits should be enforced server-side |

**Caveat:** if status is `WaitingForNextIteration` and `minimal_round_interval` is long (12h/24h), low daily count is by design. Check `round_progress` first.

## 2. Skip rate

Call `get_skip_breakdown(campaign_id, from_date, to_date)` for the audit window. The response is a list of `{ skip_reason_id, skip_reason_code, skip_reason_name, count, percentage }` sorted by count desc. Strictly aggregated by dictionary id — `percentage` always sums to ~1.0 across the rows.

Look at the dominant reason (`percentage > 0.4`) and the second-most-common reason — those two drive the fixes (mapping in `skip-reasons.md`). For ContentFilter/ProfileFilter dominance, also call `get_campaign_assistant_config(campaign_id)` so you can quote the current configuration in the report.

If the response is empty, no rows had a `SkipReasonId` set — that means either no skips occurred, or the only skips are legacy rows without a dictionary id (legacy data is excluded by design). In either case, skip rate is not a problem; move on.

## 3. Budget burn

```
daily_cost = (today_comments × comment_price) + (today_premium_comments × premium_comment_price)
            + (search_actions × search_price) + (filtered × content_filter_price + profile_filter_price)
runway_days = balance_usd / daily_cost
```

| Runway | Verdict |
|---|---|
| > 30 days | ✅ Healthy |
| 14 — 30 days | ⚠️ Plan refill |
| < 14 days | 🔴 Refill now or pause some campaigns |

If `monthly_usd` > 0, treat it as a recurring allowance and check whether daily burn × 30 fits inside it.

## 4. AI method

| Method | When appropriate |
|---|---|
| `Pro` | **The expected default for every campaign.** Research-backed, highest engagement. |
| `Common` | Only if the user explicitly chose it. Lacks Pro's research-backed quality. |
| `Off` | Lurking / monitoring only — no commenting. Almost never right for a visibility goal. Flag as 🔴 if the user's goal involves commenting. |
| `Custom` | Never use it (UI-only). |

Any campaign on `Common` or `Off` → ⚠️ "move to Pro" action item (Pro is the default method).

## 5. Schedule sanity

If `is_using_schedule=true`:

```
active_minutes_per_day = sum over schedule entries of (end_time - start_time) in minutes
max_possible_posts_per_day = active_minutes_per_day / pause_between_posts_in_minutes
```

If `max_possible_posts_per_day < post_limit_per_day` → 🔴 the limit is unreachable by construction. Either widen the schedule, lower the global pause, or lower the per-day limit to match.

## Final verdict

- All five ✅ → Healthy
- Any 🔴 → Broken (highlight at top of report)
- Otherwise → Needs attention

The verdict drives action item priority.
