# Audit Report Template

Use this exact structure. Fill placeholders, drop sections that don't apply, but don't add ones that aren't here.

---

## TL;DR

- **Biggest win:** [one line]
- **Biggest problem:** [one line]
- **Top action:** [one specific thing the user should do today]

## Campaign Health

| Campaign | Type | Today's Comments / Limit | Status | Verdict |
|---|---|---|---|---|
| [name] | ByKeyword / ByProfile | 23 / 30 | Started | ✅ Healthy |
| [name] | ByKeyword | 4 / 50 | Skipped (mostly) | 🔴 Broken |

## Account Findings

- **Balance:** $X.XX USD, monthly allowance $Y.YY
- **Burn rate:** $Z.ZZ/day at current pace → **N days of runway**
- **Global pause:** P minutes between posts → max theoretical throughput across all campaigns is M comments/day
- [Any cross-campaign issues, e.g. shared LinkedIn account throttling]

## Per-Campaign Detail

### [Campaign Name 1] — 🔴 / ⚠️ / ✅

- **Throughput:** X/Y comments today (Z%)
- **Dominant skip reason:** [from skip_reasons or "unknown — endpoint missing"]
- **AI method:** [Off/Common/Pro] — [Pro = appropriate / Common or Off = recommend moving to Pro]
- **Schedule:** [feasible / unreachable]
- **Findings:**
  - [bullet]
  - [bullet]

(Repeat for each campaign)

## Top Engagement (last 7 days)

The top 3 AI comments by total engagement (likes + replies). Help the user see *what's working* in their own voice.

1. **[post URL]** — {N} likes, {M} replies — "{first 80 chars of comment}…"
2. ...

If there were no comments over the window, omit this section.

## Action Items

Priority-ordered. For each: which campaign, what change, why. **Write every action item in plain language — no raw field names, no MCP call syntax.** **Propose only — do not execute.** Ask the user for go-ahead before applying.

1. **[Campaign X]** — raise the daily comment limit from 50 to 75. Currently hitting the cap every day; budget supports more.
2. **[Campaign Y]** — remove 3 keywords that produced 90%+ off-topic skips: "<keyword 1>", "<keyword 2>", "<keyword 3>".
3. **[Campaign Z]** — widen the maximum post age from 30 to 60 days. "Posts too old" is the dominant skip reason.
4. **[Account-level]** — top up the balance; current runway is N days.

> Internal (agent-only — never echo to the user): the corresponding tool calls are `update_campaign({post_limit_per_day})`, `modify_campaign_targets({ids_to_remove})` with `target_id` from `get_per_target_performance` / `get_campaign_targets`, and `update_campaign_assistant({post_filter_patch.skip_posts_older_than_days})`. Look them up when the user approves an action — never paste them into the report itself.

---

**Tone notes:**
- Be quantified, not vague ("73% of skips were off-topic" not "lots of skips")
- Lead with problems, end with actions
- Don't pad. If a campaign is healthy, one line is enough.
- **No raw field names anywhere the user reads** — "the daily comment limit", "off-topic skips", "Pro", "the maximum post age" — not `post_limit_per_day`, `content_filter`, `premium`, `skip_posts_older_than_days`.
