# Limits, AI Methods, Ramp-Up, and Schedule

## AI methods (from `list_lookup(kind: "AiMethod")`)

| Method | Code | When to use | Cost source |
|---|---|---|---|
| Pro | `premium` | **Default for every campaign.** Performs background research (fresh news + analytics on the post topic) and writes a fact-checked, evidence-backed comment with citations. This is "pro-commenting" — average engagement (likes + replies) is multiple times higher. Also **unlocks `opener_lines`** in `update_campaign_assistant.commenting_patch`. | `prices.premium_comment` |
| Common | `common` | Only if the user explicitly asks for it. Generates a comment from the post + persona context only, no external research. | `prices.comment` |
| Off | `off` | Likes-only campaigns. Confirm intentional. | produces no AI comments |
| Custom | `custom` | **Never select.** UI-only webhook; cannot be configured via MCP and is billed at `prices.premium_comment` (the Pro price), not `prices.comment`. If the user asks, explain it is UI-only and offer Pro/Common. | `prices.premium_comment` |

Default: **Pro**. Set Pro by default for every campaign and during setup — do not stage through Common. Use `Common` only on explicit user request. Never select `Custom`.

## Limits — UI defaults

These are what `create_campaign` applies when no patch is provided. Match these unless the user has a reason to deviate. **MCP validator now enforces UI caps strictly** — anything above the Cap column is rejected.

| Field | Default | Cap (UI) | Rationale |
|---|---|---|---|
| `post_limit_per_day` | 10 | 200 | Conservative start. Daily budget grows automatically via ramp-up unless the user disabled it. |
| `post_limit_per_profile_per_day` | 1 | 10 | Never comment on the same person twice in a day — looks spammy. |
| `post_limit_per_profile_per_week` | 7 | 50 | Default leaves room for daily engagement on the same person. |
| `post_limit_per_profile_per_month` | 30 | 150 | Prevents "stalking" appearance over time. |
| `skip_posts_older_than_days` | 90 | 90 | Default in `update_campaign_assistant.post_filter_patch`. |
| `auto_comment_enabled` | false | — | Off by default. Ask the user before turning on (Step 11 of the skill). |
| `auto_like_enabled` | false | — | Off by default. Ask the user before turning on (Step 11 of the skill). |

If the user wants to push beyond these, the patch will be rejected by the validator. Tell them MCP caps = UI caps; talk through alternatives (more campaigns, raise the LinkedIn account count, use Pro for higher engagement per comment). Volume isn't the goal — visibility on relevant posts is.

## Ramp-up (`update_campaign_assistant.ramp_up_patch`)

Ramp-up grows `post_limit_per_day` automatically over time. Created enabled by default. Tweak only on request.

| Field | Default | Range |
|---|---|---|
| `is_enabled` | true | — |
| `frequency_in_days` | 4 | 1–30 — every N days the daily limit goes up by `increment` |
| `increment` | 2 | 1–50 — comments added per ramp-up tick |
| `max_limit` | 50 | 1–200 — the highest `post_limit_per_day` the ramp-up will reach |

Once `post_limit_per_day` reaches `max_limit`, ramp-up stops growing the limit.

To turn ramp-up off: `ramp_up_patch={ is_enabled: false }`. The current `post_limit_per_day` is preserved.

## Round interval (from `list_round_intervals`)

Applies only when `is_using_schedule=false`. The interval is the *minimum* gap between consecutive rounds — actual cadence may be longer if the previous round took time.

| Interval | When to use |
|---|---|
| `AsSoonAsPossible` (0h) | Freshness-critical campaigns, e.g. trending-topic commenting. Higher LinkedIn detection risk. |
| `Every1Hour` / `Every2Hours` | Aggressive but reasonable for high-volume mature campaigns. |
| `Every4Hours` / `Every6Hours` | **Safe default** for most campaigns. |
| `Every12Hours` / `Every24Hours` | Low-volume audit/monitoring posture, or running on a tight budget. |

When `is_using_schedule=true`, the interval is ignored — see the schedule semantics below.

## How the schedule actually works

When `is_using_schedule=true`, the campaign uses the list of `{ day_of_week, start_time, end_time }` windows you sent.

**One round per window**:
1. At the start of the next window, the campaign picks a random time inside that window.
2. At that time, it runs **one** search round (yields up to `post_limit_per_day - already_done_today` posts).
3. After the round, it picks the next window (next day, or later in the same day if there is one) and picks another random time inside it.

So the number of windows per day is the rough cap on rounds per day, not the active-minute count. Examples:

- Schedule mon–fri × 1 window (09:00–17:00) → 5 rounds per week, each at a random time in the window.
- Schedule mon–fri × 2 windows (09:00–12:00 and 14:00–17:00) → 10 rounds per week, two per day.

If the user wants more posts per day than one round can yield, add more windows instead of widening one window.

`pause_between_posts_in_minutes` from `get_global_settings` only affects how comments inside a single round are paced (so they're not all posted at once); it does not increase or decrease the rounds-per-day count under a schedule.

## Cost estimation (internal reference — shown to the user ONLY at the final review)

This formula is for the agent's own calculation. **Never quote price or cost to the user during setup.** Pricing is presented exactly once — at Step 12, right before launch — and there it must be the *combined* daily spend of all running campaigns plus the new one, plus how many days the balance (monthly plan budget until `balance.monthly_usd_renews_at`, then the `balance.usd` top-up) keeps them running.

From `get_user_stats.prices`, you get per-action prices. Per-campaign daily estimate:

```
estimated_daily_cost =
    post_limit_per_day × prices.premium_comment            (Pro is the default method)
    + expected_searches × prices.search
    + expected_content_filter_evaluations × prices.content_filter
    + expected_profile_filter_evaluations × prices.profile_filter
```

where `expected_searches = target_count × rounds_per_day` (one search is charged per keyword/profile target per round). The content filter is charged per evaluated post when it is enabled; the profile filter is charged per evaluated post when geo is configured (see Step 6 of the setup skill — enabling a profile filter also forces a profile parse and adds latency). Searches and filter evaluations aren't directly observable client-side, so present the total as a range, not a precise number.
