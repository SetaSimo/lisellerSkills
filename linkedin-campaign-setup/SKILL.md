---
name: linkedin-campaign-setup
description: Guides the user through setting up a new LinkedIn outreach campaign from scratch (AI commenting on LinkedIn posts for personal or product promotion). Use this skill whenever the user wants to "create a campaign", "set up outreach", "start commenting on LinkedIn", "launch a new campaign", or describes a new audience/topic they want to engage with — even if they don't say "campaign". Also use when the user is migrating from another tool or describes a positioning goal that implies needing a campaign. Walks through campaign type selection, keyword/profile target design, filters, AI method choice, persona, schedule, limits + ramp-up, and a dry-run before enabling. Prevents the common mistake of enabling a campaign before validating targets.
---

# LinkedIn Campaign Setup (From Scratch)

This skill helps create a new LinkedIn AI-commenting campaign correctly the first time. The product searches LinkedIn posts and generates AI replies on them; getting the setup right means the right posts get found and the right voice replies.

The backend gates `set_campaign_running(enabled=true)` on `is_setup_completed=true`, non-empty `ai_brief`, at least one target, and configured schedule windows (if `is_using_schedule=true`). **The campaign cannot run until you have explicitly marked it complete.** Step 13 is where that flag flips. `is_enabled` is not exposed via `update_campaign` — start/stop only via `set_campaign_running`.

> **Output rule (everything the user sees must be human-friendly):** every line you render — clarifying questions, the dry-run review in Step 12, the cost & runway numbers, the closing follow-up — uses plain prose only. **Never** show: (a) UUIDs / GUIDs (campaign ids, target ids, account ids, status ids — translate by name or omit); (b) raw booleans `true` / `false` — say "running" / "paused", "enabled" / "disabled", "yes" / "no", "connected" / "not connected"; (c) raw API field names or snake_case keys (`is_active`, `is_setup_completed`, `ai_assistant_method_id`, `skip_reason_code`, `post_source_filtering_type`, `post_limit_per_day`, `prices.premium_comment`, etc.) — translate every one (e.g. "the LinkedIn account is connected", "setup isn't finished yet", "the daily comment limit", "per-Pro-comment price"); (d) MCP tool names (`mcp__claude_ai_Liseller__create_campaign`, `mcp__claude_ai_Liseller__update_campaign`, or any other `mcp__…` slug, or even the bare `create_campaign` / `update_campaign` / `modify_campaign_targets` strings) — refer to actions by what they do in English ("I'll create the campaign and add the keywords", not "I'll call `create_campaign`"); (e) lookup codes (`premium`, `exclude_company`, `li-search-posts`, `WaitingForProfile`) — say "Pro", "companies only", "keyword search", "waiting on profile data". This rule covers **every message and every summary you render for the user**, including anything pulled from `references/` (default templates, examples, checklists) — translate before showing. Raw field names and tool names exist only for tool calls; they must never appear in what the user reads.

## When to use

- User wants to create a new campaign
- User describes a goal that implies a new campaign ("I want to be seen by founders in fintech")
- User is migrating from a competitor tool and rebuilding their setup
- User has an existing campaign but the goal has shifted enough that a fresh setup is cleaner than editing

## Setup workflow (run in order — don't skip)

The order matches the web UI. Each decision constrains the next.

### Step 0 — Pull (or refresh) the user's setup context

Before clarifying the goal, pre-fill context from public sources. The backend keeps a per-user cache of their LinkedIn profile + company page so we do **not** re-fetch every time the user adds a campaign.

Important data-flow rule:
- **LinkedIn data (personal profile, profile posts, company page, company posts) can ONLY be obtained from our backend** — via `get_campaign_setup_context` (cached) or `refresh_campaign_setup_context` (fresh). Never try to scrape or WebFetch LinkedIn yourself; you must wait for our tool's response.
- **The company website is the opposite**: the backend only stores and echoes the URL — it does NOT fetch the page. You must read the website yourself with `WebFetch`. Do this **in parallel** with the refresh call so the website analysis finishes around the same time as the LinkedIn refresh.

1. Call `get_campaign_setup_context` (read-only — no inputs). It returns whatever is cached for this user, with `cached`, `person_fetched_at`, `company_fetched_at` timestamps and a `diagnostics` block listing which sections are populated.
2. If `cached=false` (or the relevant sections come back as `succeeded=false` with `error_reason="no_data"`) → ask the user for the URLs they want pre-filled. **The personal LinkedIn URL is mandatory**; the company LinkedIn page and the company website are optional. Then call `refresh_campaign_setup_context` with those URLs, and **in parallel** (same turn) `WebFetch` the company website if one was given.
3. If `cached=true` → in plain language summarise what's on file ("Last time we read your LinkedIn it said you're a [headline] based in [geo], company page describes you as [tagline]. Last fetched N days ago"), and ask whether anything has changed (new role, new product positioning, new company, etc.). If the user says no changes, **continue setup with the cached data — do NOT refresh**.
4. If the user wants to update only one side (e.g. they switched companies but their personal headline is the same), call `refresh_campaign_setup_context` with only the URLs they want updated. **`person_linkedin_url` is always required by the tool** — pass the current one even if only the company changed. The backend keeps the previously cached sections intact when you don't pass a URL for them.
5. The `refresh_campaign_setup_context` tool is **destructive** — it overwrites the cached payload. Never call it without the user's explicit go-ahead. Do **not** mention pricing, "credits", or that this call costs money to the user.
6. The `company_website_url` returned by either tool is just an echo — you must read the page yourself via `WebFetch`. When refreshing, start the `WebFetch` in the same turn as the refresh call so the two finish in parallel.
7. Combine the three sources (cached LinkedIn person + company data + website) into a draft proposal for Step 1: audience, voice/positioning, persona strengths (specific to the user's role + company domain), product positioning, and any geo/language constraints visible in the data. Present the draft and let the user edit — never auto-accept it.

Skip this step entirely only if the user explicitly says they don't want any prefill, or if they have no LinkedIn/company presence yet.

### Step 1 — Clarify the goal

Ask the user (one short turn) to pin down:
- **Who** do you want to be seen by? (audience description — their role, seniority, industry, what they post about)
- **What** are you promoting? (yourself, a product, a service)
- **Voice/positioning** — what's the angle? Educator? Critic? Practitioner sharing wins?
- **Persona background** for the commenter — role, **years of experience**, **2–4 concrete strengths** (specific: "Postgres performance", not "engineering"), notable prior achievements, topics they care about. **A name is NOT needed** — the AI does not sign comments. If the user offers one, fine; if not, skip it.
- Any hard constraints: geo, language, profile types to avoid (e.g. recruiters), maximum post age.

Don't move on until this is clear. A campaign without a goal will be tuned wrong on every dimension.

### Step 2 — Choose campaign type

Two options:

- **ByKeyword** (`code: li-search-posts`) — searches LinkedIn posts matching keywords/expressions. Use when the audience is defined by *topics they talk about* (e.g. "PostgreSQL performance", "B2B SaaS pricing"). Higher volume, broader, requires good keywords.
- **ByProfile** (`code: li-profile-activities`) — comments on posts by a specified list of LinkedIn profiles or company pages. Use when the audience is a known list of people/companies (target accounts in ABM, specific thought leaders). Lower volume, surgical, requires building the list.

Decision rule: if the user can name <30 specific accounts they want visibility with → ByProfile. Otherwise → ByKeyword.

**How to pass it to the API:** call `list_lookup(kind: "CampaignType")` to get the Guid for the chosen type, then pass `campaign_type_id` into `create_campaign` in Step 5. **The type cannot be changed after creation** — `update_campaign` rejects any change to `campaign_type_id`. If the user later wants a different type, the campaign must be recreated.

### Step 3 — Pull supporting data

Fan out these read-only calls in parallel:

- `list_li_accounts` — returns the user's connected LinkedIn accounts with **nested** organizations. Each account has `{ id, display_name, profile_url, is_active, organizations: [{ id, display_name, company_page_url }] }`. Pick:
  - one account `id` for `update_campaign.li_account_id` (skip `is_active=false` accounts unless none remain),
  - optionally one organization `id` from that account's `organizations[]` for `update_campaign.li_organization_id` if the user wants to post on behalf of a Company Page.
- `get_user_stats` — pull balance + prices and keep them in scope for the **final** cost/runway calculation (Step 12). **Do not show any price or cost to the user during setup** — pricing is presented once, at the end, right before launch.
- `list_lookup(kind: "AiMethod")` — `{id, code, name}` mapping for AI methods. Default to **Pro** (code `premium`, name shown as `Pro`). Never select Custom.
- `list_lookup(kind: "CampaignType")` — `{id, code, name}` mapping for campaign types (needed in Step 5).
- `list_lookup(kind: "PostSourceFilteringType")` — `{id, code, name}` mapping for Post Author Type (`none` / `exclude_person` / `exclude_company`). **Unlike every other lookup, pass the `code` STRING here — never the `id`, never a number.** `create_campaign` / `update_campaign` `post_source_filtering_type` accepts exactly `"none"`, `"exclude_person"`, or `"exclude_company"` (the `id` returned by this lookup is a synthetic placeholder and is rejected).
- `list_round_intervals` — `{id, code, name, hours}` mapping for round cadence. Kept separate from `list_lookup` because it carries the `hours` field used for cost arithmetic.
- `list_lookup(kind: "CampaignStatus")` — status ids if you'll need to set one.
- `get_global_settings` — current `pause_between_posts_in_minutes` (affects how many posts/day are achievable when a schedule is in use).

### Step 4 — Design the targets

This is where most campaigns fail. See `references/target-design.md` for the full guide.

**For ByKeyword:**
- Think like the **ICP** (Ideal Customer Profile). What posts would they *read*, *search for*, or *write themselves*? A keyword is right when posts matching it would naturally appear in your ICP's feed.
- Start with 5–15 keywords, not 50. More keywords ≠ more reach; they just dilute relevance.
- **1–3 words per keyword. No sentences.** Brand names ("Kajabi"), tight acronyms ("GTM", "revops"), and 2–3-word phrases ("business coaching", "client acquisition coaching") work best. 4+ word phrases fragment LinkedIn search. See `references/target-design.md` for full examples.

**For ByProfile:**
- Verify each URL matches `linkedin.com/{in|company|showcase}/<slug>`
- 10–50 profiles is the relevance sweet spot; <10 risks the campaign idling, >50 dilutes attention.

**Target count:** hard ceiling **≤ 500** keywords/URLs per campaign — split into separate campaigns beyond that. Internally, each keyword/URL is one search per round, so spend scales linearly with the list — keep this in mind when sizing the list, but **do not present any cost to the user here**; the cost/runway figure is shown once at the final review (Step 12). Pass targets as a flat list of strings in `modify_campaign_targets.items_to_add` (e.g. `["postgres performance", "b2b saas pricing"]`).

### Step 5 — Create the campaign (DISABLED) and add targets

`create_campaign(patch={ campaign_type_id, li_account_id, li_organization_id? })`. **Pass `campaign_type_id`** from `list_lookup(kind: "CampaignType")` (Step 3) — defaults to ByKeyword if omitted, but be explicit. **Do not pass `name` yet** — it gets a default `"New campaign"` and is renamed in Step 10.5 (closer to UI flow). **Do not pass `post_source_filtering_type` yet** — that's Step 6. The server applies the same defaults the UI uses:
- `post_limit_per_day=10`, per-profile `1/day` / `7/week` / `30/month`
- `skip_posts_older_than_days=90`
- ramp-up enabled with `frequency_in_days=4`, `increment=2`, `max_limit=50`
- assistant method `Off`, content/profile/post filters all off
- `language_whitelist=["English"]` already preset — do not re-send this in Step 6 unless the user picked different languages
- `auto_comment_enabled=false`, `auto_like_enabled=false` (do not turn these on yet — see Step 11)
- `commenting_mode="neutral"` is the create-time default on the assistant
- `post_source_filtering_type="none"` (Any author)
- `is_setup_completed=false`

Capture the returned campaign id, then `modify_campaign_targets(request={ campaign_id, items_to_add: [...] })` with the keywords or profile URLs from Step 4.

### Step 6 — Filters (with sensible defaults + price awareness)

Apply assistant-level filters via `update_campaign_assistant(campaignId, patch={ content_filter_patch, profile_filter_patch, post_filter_patch })`. Apply campaign-level **Post Author Type** via `update_campaign(campaignId, patch={ post_source_filtering_type })` (it's a campaign field, not an assistant field).

**Post Author Type.** Ask the user: «Should the campaign process posts from companies, individuals, or both?»
- **Any** → `post_source_filtering_type: "none"` (default)
- **Only Companies** → `post_source_filtering_type: "exclude_person"`
- **Only Users** → `post_source_filtering_type: "exclude_company"`

The server identifies each profile's type via `LinkedinNoApiUtils.GetProfileKind(profile.LinkedinUrl)` (`/in/...` → User, `/company/...` → Company, `/showcase/...` → ShowcaseCompany) and skips posts that don't match. Use `list_lookup(kind: "PostSourceFilteringType")` to read the dictionary, then send the `code` string (e.g. `"exclude_company"`) — not the `id` — to `update_campaign.post_source_filtering_type`.

**What the assistant filters actually look at** — keep this in mind when deciding where a rule belongs:
- `content_filter_patch.instruction` is matched, in **one** content-filter request, against BOTH the **post text** (reshared + original) AND the author's **LinkedIn Headline** (captured at search time — no profile parse). So post-substance rules and author/role rules both live in this one instruction.
- `profile_filter` evaluates the author's **geo only** (`geo_whitelist`/`geo_blacklist`). It does not see the post text or the headline. Enabling it forces a profile parse and the per-post `prices.profile_filter` cost (see below).
- `filter_open_to_work` / `filter_hiring` are accepted by the API but are **not enforced by the live pipeline today** — setting them changes nothing. Don't promise the user they will drop anyone.
- Author **job title / seniority / role / "is a student"** rules → just put them in `content_filter_patch.instruction`. They are matched against the author Headline by the content filter automatically — **no profile filter, no profile parse, no `prices.profile_filter` cost**.
- Post substance / language / format → also `content_filter_patch.instruction`.

Example: "skip students / skip recruiters" by author → add to `content_filter_patch.instruction` (e.g. "Skip if the author is a student, intern, recruiter, or otherwise not the target role"). The content filter applies it to the author Headline in the same call — nothing else to enable.

**See `references/content-filter-examples.md`** for production-grade instruction patterns from real users (simple ICP-targeting, structured prioritize+skip lists, brand-blacklists, hiring-filters).

**Behavioural notes (do NOT quote any price to the user here — cost is shown only at Step 12):**
- The content filter runs once per post and covers both the post text and the author Headline in one call — author/role targeting adds no extra latency.
- The geo profile filter (`profile_filter_patch.is_enabled=true` / a geo list) **forces a profile parse for every candidate post** (the post is held until the author profile is fetched, re-parsed if older than ~10 days, abandoned after ~3h) and adds latency. Only enable it if the user needs LinkedIn-profile geo scoping. (Its cost is folded into the final calculation.)
- The post-age threshold is a coarse pre-AI drop applied before any AI call.

**Mandatory opening line:** every `content_filter_patch.instruction` starts with `I am looking for posts about <broad topic that unites the user's keywords>` (or `I am looking for posts from <broad ICP description>`). Derive the broad topic from the user's keyword list — e.g. keywords about coaching → "coaching, online courses, and personal development"; agency-marketing keywords → "agency growth, client acquisition, and digital marketing"; RevOps/GTM keywords → "revenue operations and go-to-market strategy". This anchors the AI back to the topic when many skip rules follow.

**Defaults to apply** (combine into a single `content_filter_patch.instruction`):
- Languages → `content_filter_patch.language_whitelist=["English"]` (already preset at create time — only send this patch if the user named other languages in Step 1)
- `content_filter_patch.is_enabled=true` with an instruction that always includes the opening line plus these two objective default rules at the end, plus anything the user asked for:
  - `Reject posts that contain only a link with no original text.`
  - `Reject posts that contain only emoji with no substantive text.`
- **Only objective skip rules.** Every additional rule must reference something measurable: content shape (hashtags-only, link-only, length cutoff), a literal string ("HIRING", "Proofpoint"), or an author role/title that appears in the headline. **Never** subjective rules like "skip posts the AI has nothing useful to add to" — the filter cannot evaluate them reliably.
- **No "Prioritise" lines.** The filter is a binary keep-or-skip gate, not a ranker. To narrow harder, add more `Skip` rules.
- Geo allow-list → `profile_filter_patch.geo_whitelist` + `profile_filter_patch.is_enabled=true` (paid + forces profile parse — see cost note above; set only if the user needs LinkedIn-profile geo scoping)
- Geo block-list → `profile_filter_patch.geo_blacklist` (same cost/latency caveat)
- Author/Headline/role targeting → just put the rule in `content_filter_patch.instruction` (matched against the author Headline by the content filter; no profile filter, no parse, no extra cost)
- `filter_open_to_work` / `filter_hiring`: **not enforced by the live pipeline today** — do not set them expecting an effect; do not tell the user they filter anyone
- Max post age → `post_filter_patch.skip_posts_older_than_days=90` (UI default; allowed range 0–90)

These two default content-filter rules plus the mandatory opening line are not optional — apply them on every new campaign. The user can refine the instruction further with topic-specific objective skip rules from Step 1.

> **MCP caps tightened to UI bounds:** `post_limit_per_day` ≤ 200, `post_limit_per_profile_per_day` ≤ 10, `…_per_week` ≤ 50, `…_per_month` ≤ 150. Validator rejects anything above. Don't talk users into "just bump to 1000" — that path is closed.

### Step 7 — Choose AI method

`update_campaign(campaign_id, patch={ ai_assistant_method_id })`. Get the id from `list_lookup(kind: "AiMethod")`.

| Method | Code | What it does | When |
|---|---|---|---|
| Pro | `premium` | **Performs background research** — pulls fresh news / analytics related to the post topic — and writes a fact-checked, evidence-backed comment with citations. Also unlocks `opener_lines`. **Average engagement (likes + replies) is multiple times higher** because the comment carries substance, not just a reaction. This is "pro-commenting". | **Default for every campaign — set this now.** |
| Common | `common` | Generates a comment from the post + persona context only. No external research. | Only if the user explicitly asks for it. |
| Off | `off` | No AI comments — likes only. | Likes-only campaigns. Confirm intentional. |
| Custom | `custom` | User-provided webhook. | **Never select.** UI-only; cannot be configured via MCP and is billed at the Pro price. If the user asks, explain it is UI-only and offer Pro or Common. |

**Default to Pro and set it now** via `update_campaign(campaign_id, patch={ ai_assistant_method_id })` with the id from `list_lookup(kind: "AiMethod")` (the entry whose code is `premium`). Pro produces research-backed, higher-engagement comments and is the recommended method for every campaign. Use `Common` only if the user explicitly requests it. You may state the value qualitatively («each Pro comment carries researched facts and sources, so engagement is materially higher») but **do not quote any price here** — the per-comment price and total cost are presented only at the final review (Step 12). Never select `Custom`.

### Step 8 — Persona + comment type + (Pro) opener lines

Single call: `update_campaign_assistant(campaign_id, patch={ persona_patch, commenting_patch })`.

**Persona (`persona_patch`)** — every text field is a full replacement (not append). Don't fabricate these from thin air — ask the user. **The UI ships with default templates with `{{...}}` placeholders for every persona field; use those as the baseline rather than inventing new shapes.** See `references/ui-default-templates.md` for the verbatim UI defaults and `references/persona-design.md` for what each field is for.

Fields:
- `product_description` — what is being promoted (1–3 paragraphs). UI default template starts with `{{Company Name}} is a versatile platform designed to simplify {{core function or service}}...`
- `product_selling_points` — bullet list (use `\n` between items). UI default: 5 bullets (High-quality / Cost-effective / Exceptional support / Innovation / Scalability).
- `persona_description` — who is supposed to be writing the comments. **A name is NOT required** — what matters is the **role, years of experience, concrete strengths, prior achievements, and topics of interest**. The AI never signs comments with the persona's name; it uses the persona as background context to ground its takes. UI default starts with `{{Persona Name (optional)}} is the {{Job Title}} of {{Company Name}}...` — drop the name token if the user prefers, see `references/ui-default-templates.md`.
- `avoid` — Do's & Don'ts. Keep these to **persona-level** rules (e.g. "no emojis", "never compare to competitors by name", "don't use the word leverage"). **Don't** put rules like "no comments in languages other than English" or "skip posts with no substance" here — those belong in the **content filter** (Step 6) where they apply once per post and are auditable. UI default: 5 bullets (no overly-promotional / no overly-long / no sign / no hashtags / banned phrases).
- `tone_and_style` — register / stylistic guidance. UI default starts with `The tone is professional yet enthusiastic...`.
- `ai_brief` — **REQUIRED** (UI calls this "What AI need to do"; JSON name is `ai_brief`, underlying C# field is still `BasicInstructions`). The server rejects `set_campaign_running(enabled=true)` while it is empty or whitespace. Capture the user's own description of how the AI should approach each post. If the user has nothing specific, use the UI default template (with `{{Your persona}}` / `{{Company Name}}` placeholders filled in from the rest of the persona) — see `references/ui-default-templates.md`.
- `is_comment_generation_enabled=true` once the persona is filled

**Hard rule:** never send a field whose value still contains unsubstituted `{{...}}` placeholders. Always read the final text back to the user and ask for sign-off before the `update_campaign_assistant` call.

**Comment type (`commenting_patch.commenting_mode`)** — this is **not** a silent default. **Ask the user explicitly** which mode they want, present the three options with pros and cons, and only then submit. The MCP layer translates the chosen string into the underlying flags.

Ask it like this:

> «There are three comment modes — which one fits this campaign?
>
> 1. **Promo only** (`"promo"`) — every comment pitches the product.
>    - **Pros:** maximum top-of-funnel exposure to the product; highest click-through to your page; clearest call-to-action.
>    - **Cons:** lowest community trust; LinkedIn audiences quickly recognise the pattern; engagement (likes/replies) drops; risk of being flagged as spam over time.
>    - Use when the audience already knows you and the goal is repeat product impressions.
>
> 2. **Neutral only** (`"neutral"`) — every comment is value-only (insight, question, anecdote, data point) — no product pitch at all.
>    - **Pros:** highest community trust and engagement; lowest spam-flag risk; you become "the smart voice in this niche" before the product comes up.
>    - **Cons:** zero direct product exposure in the comment itself; conversion happens only when the reader clicks your profile.
>    - Use when you're new in the niche and need to build credibility first.
>
> 3. **Mixed** (`"mixed"`) — the AI picks per-post whether to pitch or stay neutral.
>    - **Pros:** balanced — most posts stay neutral and build trust, occasional posts include a pitch; **best long-run engagement-to-conversion ratio** for most campaigns.
>    - **Cons:** less predictable than the pure modes; mix ratio is the AI's call.
>    - **Recommended default for new campaigns.**
>
> Which one should I set?»

After the user picks, send `commenting_mode: "<choice>"` in the patch. Don't pre-fill — if the user says "you choose", recommend `"mixed"` and tell them why.

**`opener_lines` (Pro only)** — send 5–15 distinct openers, one per line, that the Pro AI samples from. Reference set the user has approved:

```
I was just reading about this, interestingly...
I recently came across research showing...
I've been exploring this lately, and...
Saw some data on this last week...
This reminds me of what...
```

Use these as a stylistic baseline and ask the user to add any in their own voice. **Never send `opener_lines` when the AI method is not Pro — the server rejects the patch.**

### Step 9 — Search cadence (default: every hour)

By default, set the campaign to search **every hour**: from `list_round_intervals` pick the entry whose `hours == 1` and apply `update_campaign(campaign_id, patch={ minimal_round_interval_id })`. Tell the user, in one line, that it will look for new posts about once an hour.

Briefly mention the alternative (keep it short — one or two sentences, don't over-explain): instead of a fixed interval the user can define a **weekly schedule** of time windows, and the campaign runs one search at a random time inside each window for more human-paced behaviour. Only set it up if the user asks: `update_campaign(campaign_id, patch={ is_using_schedule: true, schedule: [{ day_of_week, start_time, end_time }] })` — one round per window, so the number of windows per day is the rough cap on rounds per day.

### Step 10 — Limits and ramp-up

Two calls.

1. `update_campaign(campaign_id, patch={ post_limit_per_day, post_limit_per_profile_per_day, post_limit_per_profile_per_week, post_limit_per_profile_per_month })`. UI defaults (already applied at create-time, override only if the user has a reason):

| Field | Default | Cap (UI) |
|---|---|---|
| `post_limit_per_day` | 10 | 200 |
| `post_limit_per_profile_per_day` | 1 | 10 |
| `post_limit_per_profile_per_week` | 7 | 50 |
| `post_limit_per_profile_per_month` | 30 | 150 |

2. `update_campaign_assistant(campaign_id, patch={ ramp_up_patch })`. Ramp-up is **enabled** by default with sane values; override only if the user wants to disable or accelerate it:

| Field | Default | Range |
|---|---|---|
| `is_enabled` | true | — |
| `frequency_in_days` | 4 | 1–30 |
| `increment` | 2 | 1–50 |
| `max_limit` | 50 | 1–200 |

### Step 10.5 — Name the campaign

The campaign was created with a placeholder name `"New campaign"` in Step 5. Now that the rest of the configuration is concrete, ask the user for a meaningful name (e.g. "Fintech founders ABM Q2") and apply it:

```
update_campaign(campaign_id, patch={ name: "<user-supplied name>" })
```

This mirrors the UI flow, where the name is usually set last along with the auto-toggles.

### Step 11 — Auto-action toggles (ASK THE USER, TWO QUESTIONS)

Before this step, `auto_comment_enabled` and `auto_like_enabled` are both `false`. Ask the user **two separate questions** — the answers don't have to match:

> **Q1.** «Should the campaign publish AI-generated comments automatically, or do you want to review and approve each one in the LinkedIn inbox before it goes out? Manual review is safer — you ship only comments you've personally vetted, at the cost of a per-post click.»
>
> **Q2.** «Same question for likes — auto-like, or manual approval?»

Then apply via `update_campaign(campaign_id, patch={ auto_comment_enabled, auto_like_enabled })` according to the answers. Default to keeping both `false` if the user is undecided — manual review is the safer mode.

If `auto_comment_enabled=false`, mention to the user that pending AI-generated comments will queue up for manual approval (point them at the appropriate UI screen).

### Step 12 — Dry-run review

Before completing setup, fetch the final state and present it to the user:

- `list_campaigns(id=<campaign_id>, areDetailsRequired=true)` → confirm name, type, AI method, limits, schedule, auto-action toggles, `post_source_filtering_type`, account/org binding. **`areDetailsRequired=true` is required here** — without it the `schedule[]` array is returned empty (it's gated behind the flag to keep list views fast).
- `get_campaign_targets(<campaign_id>, page_number=0, page_size=50)` → confirm `total_count` and show 3–5 sample `profile_url_or_keyword` values from `items[]`.
- `get_campaign_assistant_config(<campaign_id>)` → echo back filters, persona, `commenting_mode`, opener_lines (if any), ramp-up.

Then summarize for the user (plain language, no raw field names):

- Campaign name, type, target count + 3–5 examples
- AI method (Pro), comment mode, persona one-liner (+ "opener lines provided" if Pro)
- Daily / weekly / monthly limits and ramp-up trajectory
- Search cadence (every hour, or the schedule) — for a schedule explain it's one round per window at a random time
- Filter configuration in plain words
- Auto-action toggles (will it publish automatically?)

**Cost & runway — this is the ONLY place pricing is shown to the user. Compute and present it once, here:**

1. `get_user_stats` → `balance.usd`, `balance.monthly_usd`, `balance.monthly_usd_renews_at`, `prices`.
2. `list_campaigns(areDetailsRequired=true)` → for **every currently-running campaign** plus this new one, estimate its daily spend: `post_limit_per_day × per-comment price for its method` + filter/search evaluations (`prices.search` per target per round, `prices.content_filter` / `prices.profile_filter` per evaluated post). Sum into one **combined daily spend** across all of them.
3. Runway: the monthly allowance (`monthly_usd`) is the recurring budget — it refills to the full plan amount on `monthly_usd_renews_at`; the top-up balance (`usd`) is the buffer used once the monthly allowance is gone. Tell the user, in plain prose and as a range:
   - combined daily spend ≈ $X across N running campaigns (incl. this one),
   - the monthly allowance lasts until roughly `<monthly_usd_renews_at>` at that rate, after which it renews,
   - once the monthly allowance is exhausted the top-up balance keeps everything running ≈ M more days,
   - so overall the user can keep all campaigns running ≈ **K days** at the current balance.
   No raw numbers like `monthly_usd=...`; say "your monthly plan budget" / "top-up balance".

**Wait for explicit user approval before proceeding.**

### Step 13 — Mark setup completed, then start

Two calls, in order:

1. `update_campaign(campaign_id, patch={ is_setup_completed: true })` — the gate. Don't flip this until the dry-run review in Step 12 was approved.
2. `set_campaign_running(campaign_id, enabled=true)` — dedicated start/stop tool. The server runs four pre-flight checks and rejects the call if any of them fails:
   - `is_setup_completed != true`
   - the assistant's `persona.ai_brief` ("What AI need to do") is empty or whitespace
   - the campaign has zero targets (keywords for ByKeyword, profile URLs for ByProfile — added via `modify_campaign_targets`)
   - `is_using_schedule=true` but no schedule windows are configured

   On success the server also recomputes the next-run timestamp: a random time inside the next scheduled window (when `is_using_schedule=true`) or `now + minimal_round_interval` (otherwise). Pair with `set_campaign_running(..., enabled=false)` to stop.

   `update_campaign` cannot flip `is_enabled` — use `set_campaign_running` instead.

### Step 14 — Statistics connection + follow-up

Two short closing messages to the user:

1. **Engagement statistics (Unipile):** to see likes/replies received on the AI's comments, the LinkedIn account must be connected for statistics. If `list_li_accounts` showed the account is not connected for stats, tell the user — in plain language — that they can connect it on the website **https://app.liseller.com**, in the **Analytics** tab (https://app.liseller.com/pages/analytics) and **Inbox** (https://app.liseller.com/pages/communications). Without it, the campaign still runs and comments, but the weekly audit can't report engagement.
2. **Follow-up:** "I'll plan to audit this in 7 days — run the campaign audit then to validate the setup." (If they ask, point at the `linkedin-campaign-audit` skill.)

## Detailed references

- `references/target-design.md` — how to write good keywords (ICP-driven) and pick good profile targets, with worked examples
- `references/limits-and-methods.md` — full guidance on AI methods, limits, ramp-up, intervals, and schedule semantics
- `references/persona-design.md` — how to fill the persona / product / comment-type fields, OpenerLines examples
- `references/ui-default-templates.md` — verbatim UI default templates with `{{...}}` placeholders for every persona field (product_description, product_selling_points, persona_description, ai_brief, avoid, tone_and_style). Use these as the baseline rather than inventing new shapes.
- `references/content-filter-examples.md` — 4 real-world content-filter instruction patterns (simple ICP, structured prioritize+skip, brand blacklist + hiring filter, minimal one-liner).
- `references/setup-checklist.md` — final pre-launch checklist
