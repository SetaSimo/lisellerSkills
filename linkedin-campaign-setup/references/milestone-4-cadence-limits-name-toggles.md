# Milestone 4 — Search cadence, limits + ramp-up, name, auto-action toggles

Read this after Milestone 3. It covers the original Steps 9–11 (including 10.5). When you finish it,
return to the milestone map in `SKILL.md` and open the Milestone 5 file.

## Step 9 — Search cadence (default: every hour)

By default, set the campaign to search **every hour**: from `list_round_intervals` pick the entry whose `hours == 1` and apply `update_campaign(campaign_id, patch={ minimal_round_interval_id })`. Tell the user, in one line, that it will look for new posts about once an hour.

Briefly mention the alternative (keep it short — one or two sentences, don't over-explain): instead of a fixed interval the user can define a **weekly schedule** of time windows, and the campaign runs one search at a random time inside each window for more human-paced behaviour. Only set it up if the user asks: `update_campaign(campaign_id, patch={ is_using_schedule: true, schedule: [{ day_of_week, start_time, end_time }] })` — one round per window, so the number of windows per day is the rough cap on rounds per day.

## Step 10 — Limits and ramp-up

Both are already applied at create-time with UI defaults — override only if the user has a reason:

1. `update_campaign(campaign_id, patch={ post_limit_per_day, post_limit_per_profile_per_day/week/month })` — defaults 10 / 1 / 7 / 30 (caps 200 / 10 / 50 / 150).
2. `update_campaign_assistant(campaign_id, patch={ ramp_up_patch })` — ramp-up enabled by default (`frequency_in_days=4`, `increment=2`, `max_limit=50`).

Full tables + ranges in `references/limits-and-methods.md`.

## Step 10.5 — Name the campaign

The campaign was created with a placeholder name `"New campaign"` in Milestone 2. Now that the rest of the configuration is concrete, ask the user for a meaningful name (e.g. "Fintech founders ABM Q2") and apply it:

```
update_campaign(campaign_id, patch={ name: "<user-supplied name>" })
```

This mirrors the UI flow, where the name is usually set last along with the auto-toggles. **Record this name in your session ledger** (see `SKILL.md`) so you can refer to the campaign in plain language later and track whether it has been launched.

## Step 11 — Auto-action toggles (ASK THE USER, TWO QUESTIONS)

Before this step, `auto_comment_enabled` and `auto_like_enabled` are both `false`. Ask the user **two separate questions** — the answers don't have to match:

> **Q1.** «Should the campaign publish AI-generated comments automatically, or do you want to review and approve each one in the LinkedIn inbox before it goes out? Manual review is safer — you ship only comments you've personally vetted, at the cost of a per-post click.»
>
> **Q2.** «Same question for likes — auto-like, or manual approval?»

Then apply via `update_campaign(campaign_id, patch={ auto_comment_enabled, auto_like_enabled })` according to the answers. Default to keeping both `false` if the user is undecided — manual review is the safer mode.

If `auto_comment_enabled=false`, mention to the user that pending AI-generated comments will queue up for manual approval (point them at the appropriate UI screen).

---

**Milestone 4 done.** Return to the milestone map and open `references/milestone-5-review.md`.
