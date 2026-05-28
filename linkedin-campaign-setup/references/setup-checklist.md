# Pre-Launch Checklist

Run through this before flipping `is_enabled=true`. Don't skip — most setup failures are caught here.

The backend gates `set_campaign_running(enabled=true)` on (1) `is_setup_completed=true`, (2) non-empty `persona.ai_brief`, (3) at least one synced target, and (4) at least one schedule window if `is_using_schedule=true`. Until each of these holds, the campaign will refuse to start.

## Goal alignment
- [ ] User stated the audience clearly
- [ ] User stated what they're promoting
- [ ] Voice/positioning is clear enough to pick the AI method and write the persona
- [ ] Hard constraints captured (geo, language, Open-To-Work, Hiring, max post age)

## Targets
- [ ] Campaign type matches the audience definition (ByKeyword for topical, ByProfile for named accounts)
- [ ] **ByKeyword:** 5–15 keywords (relevance sweet spot), **each 1–3 words** (brand names, acronyms, short phrases — no sentences), anchored to ICP
- [ ] **ByProfile:** 10–50 URLs (relevance sweet spot), all matching `linkedin.com/{in|company|showcase}/<slug>`, mix of tiers
- [ ] Total targets ≤ 500 (hard ceiling). Every keyword/URL is one search per round — cost scales linearly with target count.
- [ ] No hashtag-style keywords; keep each keyword to 3 words or fewer
- [ ] Targets synced via `modify_campaign_targets` (flat list of strings in `items_to_add`) and confirmed

## Filters (`update_campaign_assistant`)
- [ ] Language allowlist set (`content_filter_patch.language_whitelist`) — defaults to English if user didn't specify
- [ ] Content filter enabled with: (1) mandatory opening line `I am looking for posts about <broad topic that unites the user's keywords>`, (2) the two objective default rules (skip link-only posts, skip emoji-only posts), and (3) any additional objective skip rules from the user (literal strings, content shape, role/title). **No subjective rules and no "Prioritise" lines.**
- [ ] Author/role rule ("only CTOs", "skip students", "skip recruiters") → put it in `content_filter_patch.instruction`. The content filter matches the instruction against the author Headline as well as the post text, in one call — no profile filter, no parse, no extra cost.
- [ ] Geo scoping needed? Set `profile_filter_patch.is_enabled=true` + `geo_whitelist`/`geo_blacklist`. The profile filter is geo-only; enabling it forces a profile parse per post and adds latency — only enable if the user needs LinkedIn-profile geo scoping. (No price quoted here — cost is shown only at the final review.)
- [ ] `filter_open_to_work` / `filter_hiring` are accepted but NOT enforced by the live pipeline today — do not promise the user these will drop anyone
- [ ] `post_filter_patch.skip_posts_older_than_days` ≤ 90
- [ ] No price/cost mentioned to the user anywhere in Filters — pricing appears only at the final review

## Search cadence
- [ ] Default applied: search **every hour** (`minimal_round_interval_id` = the `list_round_intervals` entry with `hours == 1`)
- [ ] Weekly schedule mentioned briefly; only configured (`is_using_schedule=true` + windows) if the user asked

## AI method
- [ ] Method set to **Pro** (the default for every campaign); Common only if the user explicitly requested it; never Off unless intentional likes-only; never Custom (UI-only, cannot be set via MCP)
- [ ] No per-comment price mentioned here — cost is presented only at the final review
- [ ] At least 5 `opener_lines` collected (Pro unlocks them; use the reference set + user-specific additions)

## Persona (`update_campaign_assistant.persona_patch`)
- [ ] `product_description` populated
- [ ] `product_selling_points` populated
- [ ] `persona_description` populated
- [ ] `avoid` populated with persona-level rules only (post-selection rules moved to content filter)
- [ ] `tone_and_style` populated
- [ ] **`ai_brief` populated** ("What AI need to do") — campaign refuses to start without this. Use the default template (substituting `{{persona_description}}` and `{{product_name}}`) if the user has no specific instructions, and confirm with the user before submitting.
- [ ] `is_comment_generation_enabled = true`
- [ ] No text was fabricated — every field came from the user (or was confirmed back)

## Commenting (`update_campaign_assistant.commenting_patch`)
- [ ] `commenting_mode` set to one of `"promo"` / `"neutral"` / `"mixed"`
- [ ] `opener_lines` provided (method is Pro by default)

## Schedule
- [ ] If `is_using_schedule=true`, the user understands "one round per window at a random time" semantics
- [ ] Window count per day matches the desired round count

## Limits + ramp-up
- [ ] `post_limit_per_day` ≤ 200 (default 10)
- [ ] Per-profile limits ≤ UI caps (10/day, 50/wk, 150/mo)
- [ ] Ramp-up `frequency_in_days` ∈ [1, 30], `increment` ∈ [1, 50], `max_limit` ∈ [1, 200]
- [ ] If ramp-up disabled, user knows the daily limit stays fixed

## Auto-action toggles (Step 11)
- [ ] User explicitly asked whether to publish AI comments automatically
- [ ] User explicitly asked whether to auto-like
- [ ] `auto_comment_enabled` and `auto_like_enabled` set per the answer (default false if undecided)

## Account context
- [ ] `li_account_id` came from `list_li_accounts` (not invented)
- [ ] `li_organization_id` (if posting from company page) came from the same response, nested under the chosen account
- [ ] User has confirmed the LinkedIn account is the right one to post from

## Targets verified
- [ ] `get_campaign_targets(campaign_id)` returns `total_count` matching what was synced
- [ ] Sample of 3–5 `profile_url_or_keyword` values shown to the user for sanity check

## Final review with user
- [ ] Showed the user the full summary in plain language (name, type, target count + examples, AI method, limits + ramp-up, cadence/schedule, filter config, persona one-liner, comment mode, auto-action toggles)
- [ ] **Cost & runway computed and shown here (and nowhere earlier):** combined daily spend of all running campaigns + the new one, and ≈ how many days the balance keeps them running (monthly plan budget until renewal, then top-up balance) — as a range, in plain words
- [ ] Unipile/statistics connection explained (engagement stats need a connected Unipile account — see Step 14)
- [ ] Got explicit approval to enable
- [ ] Set follow-up: audit in 7 days

## Mark setup completed, then enable
- [ ] Call `update_campaign(campaign_id, patch={ is_setup_completed: true })`
- [ ] Then call `set_campaign_running(campaign_id, enabled=true)`

`set_campaign_running` runs four server-side pre-flight checks and will reject the call if any fail:
1. `is_setup_completed != true`
2. `persona.ai_brief` is empty or whitespace
3. zero synced targets
4. `is_using_schedule=true` with no schedule windows configured

If any item above is unchecked, **don't flip `is_setup_completed`** — the backend will accept the patch but you'll be lying to future maintainers. Tell the user which item is missing instead.
