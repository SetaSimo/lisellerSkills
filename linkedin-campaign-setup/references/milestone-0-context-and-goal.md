# Milestone 0 — Setup context + goal

Read this when you start a new campaign. It covers the original Steps 0–1. When you finish it, you no
longer need it in focus — return to the milestone map in `SKILL.md` and open the Milestone 1 file.

## Step 0 — Pull (or refresh) the user's setup context

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

## Step 1 — Clarify the goal

Ask the user (one short turn) to pin down:
- **Who** do you want to be seen by? (audience description — their role, seniority, industry, what they post about)
- **What** are you promoting? (yourself, a product, a service)
- **Voice/positioning** — what's the angle? Educator? Critic? Practitioner sharing wins?
- **Persona background** for the commenter — role, **years of experience**, **2–4 concrete strengths** (specific: "Postgres performance", not "engineering"), notable prior achievements, topics they care about. **A name is NOT needed** — the AI does not sign comments. If the user offers one, fine; if not, skip it.
- Any hard constraints: geo, language, profile types to avoid (e.g. recruiters), maximum post age.

Don't move on until this is clear. A campaign without a goal will be tuned wrong on every dimension.

---

**Milestone 0 done.** Return to the milestone map and open `references/milestone-1-type-and-targets.md`.
