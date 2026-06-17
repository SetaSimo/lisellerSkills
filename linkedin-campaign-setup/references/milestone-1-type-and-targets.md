# Milestone 1 — Campaign type + supporting data + targets

Read this after Milestone 0. It covers the original Steps 2–4. When you finish it, return to the
milestone map in `SKILL.md` and open the Milestone 2 file.

## Step 2 — Choose campaign type

Two options:

- **ByKeyword** (`code: li-search-posts`) — searches LinkedIn posts matching keywords/expressions. Use when the audience is defined by *topics they talk about* (e.g. "PostgreSQL performance", "B2B SaaS pricing"). Higher volume, broader, requires good keywords.
- **ByProfile** (`code: li-profile-activities`) — comments on posts by a specified list of LinkedIn profiles or company pages. Use when the audience is a known list of people/companies (target accounts in ABM, specific thought leaders). Lower volume, surgical, requires building the list.

Decision rule: if the user can name <30 specific accounts they want visibility with → ByProfile. Otherwise → ByKeyword.

**How to pass it to the API:** call `list_lookup(kind: "CampaignType")` to get the Guid for the chosen type, then pass `campaign_type_id` into `create_campaign` in Milestone 2. **The type cannot be changed after creation** — `update_campaign` rejects any change to `campaign_type_id`. If the user later wants a different type, the campaign must be recreated.

## Step 3 — Pull supporting data

Fan out these read-only calls in parallel:

- `list_li_accounts` — returns the user's connected LinkedIn accounts with **nested** organizations. Each account has `{ id, display_name, profile_url, is_active, organizations: [{ id, display_name, company_page_url }] }`. Pick:
  - one account `id` for `update_campaign.li_account_id` (skip `is_active=false` accounts unless none remain),
  - optionally one organization `id` from that account's `organizations[]` for `update_campaign.li_organization_id` if the user wants to post on behalf of a Company Page.
- `get_user_stats` — pull balance + prices and keep them in scope for the **final** cost/runway calculation (Milestone 5). **Do not show any price or cost to the user during setup** — pricing is presented once, at the end, right before launch.
- `list_lookup(kind: "AiMethod")` — `{id, code, name}` mapping for AI methods. Default to **Pro** (code `premium`, name shown as `Pro`). Never select Custom.
- `list_lookup(kind: "CampaignType")` — `{id, code, name}` mapping for campaign types (needed in Milestone 2).
- `list_lookup(kind: "PostSourceFilteringType")` — `{id, code, name}` mapping for Post Author Type (`none` / `exclude_person` / `exclude_company`). **Unlike every other lookup, pass the `code` STRING here — never the `id`, never a number.** `create_campaign` / `update_campaign` `post_source_filtering_type` accepts exactly `"none"`, `"exclude_person"`, or `"exclude_company"` (the `id` returned by this lookup is a synthetic placeholder and is rejected).
- `list_round_intervals` — `{id, code, name, hours}` mapping for round cadence. Kept separate from `list_lookup` because it carries the `hours` field used for cost arithmetic.
- `list_lookup(kind: "CampaignStatus")` — status ids if you'll need to set one.
- `get_global_settings` — current `pause_between_posts_in_minutes` (affects how many posts/day are achievable when a schedule is in use).

## Step 4 — Design the targets

This is where most campaigns fail.

**For ByKeyword — see `references/target-design.md` for the full guide:**
- Think like the **ICP** (Ideal Customer Profile). What posts would they *read*, *search for*, or *write themselves*? A keyword is right when posts matching it would naturally appear in your ICP's feed.
- **Generate ~30 keywords (aim for 25–30)** yourself from the ICP, the user's profile/company context, and their stated goal — then present the full set so the user can trim or add to it. Don't make the user brainstorm from scratch.
- **1–3 words per keyword. No sentences.** Brand names ("Kajabi"), tight acronyms ("GTM", "revops"), and 2–3-word phrases ("business coaching", "client acquisition coaching") work best. 4+ word phrases fragment LinkedIn search.

**For ByProfile:**
- Verify each URL matches `linkedin.com/{in|company|showcase}/<slug>`
- 10–50 profiles is the relevance sweet spot; <10 risks the campaign idling, >50 dilutes attention.
- **If the user wants you to FIND profiles by criteria** (job title, seniority, industry, location, …) instead of pasting their own URLs, you **MUST** use the `search_profiles` tool — never guess or invent profile URLs. **Read `references/profile-search.md` before searching**: it documents every filter, the exact controlled-vocabulary values, and the discover→present→pick→add workflow. Only search when the user asks to; never for ByKeyword campaigns.

**Target count:** hard ceiling **≤ 500** keywords/URLs per campaign — split into separate campaigns beyond that. Each keyword/URL is one search per round, so spend scales linearly with the list (keep in mind when sizing — but cost is shown only at Milestone 5). Pass targets as a flat list of strings in `modify_campaign_targets.items_to_add` (e.g. `["postgres performance", "b2b saas pricing"]`). You add them in Milestone 2 right after creating the campaign.

---

**Milestone 1 done.** Return to the milestone map and open `references/milestone-2-create-and-filters.md`.
