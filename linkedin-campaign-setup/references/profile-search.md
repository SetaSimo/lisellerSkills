# Profile search (ICP discovery via `search_profiles`)

For **ByProfile** campaigns, when the user wants to *find* the right LinkedIn profiles by criteria
instead of pasting a ready list of URLs, you **MUST** use the `search_profiles` tool. Never guess,
invent, or hand-construct a profile URL, and never use this tool for ByKeyword campaigns or to look up a
single already-known person. Only search when the user actually asks you to find people.

`search_profiles` is a read-only discovery tool backed by a third-party people database. It returns up to
**50 matches per call**, each with a public LinkedIn URL ready to drop straight into
`modify_campaign_targets.items_to_add`. It saves nothing on its own.

## How it matches

- All filters are **optional** and **AND-combined** — an empty request returns broad results; every filter
  you add narrows the set. Within a single keyword field the terms are OR-matched.
- Some fields are **free text** (keywords, names). Some are **controlled vocabulary** — they only work with
  the exact values listed below; anything else is silently ignored, so a typo just drops the filter.
- The tool caps at 50 results. Prefer **several narrow searches** (e.g. one per role or per region) over one
  broad query that overflows the cap.

## Filters

**Person / role**
- `job_title_keywords` — free text, OR-matched against the current job title, e.g. `["founder", "head of growth"]`.
- `exclude_job_title_keywords` — drop profiles whose current title matches any of these.
- `seniority_levels` — **controlled vocabulary**, use only: `owner`, `partner`, `c-suite`, `vp`, `manager`, `director`, `head`.
- `job_title_include_past_experiences` — `true` also matches job-title keywords against past roles (default `false`).
- `names` — exact person names to match.
- `about_keywords` — matched against the profile's About / summary section.
- `headline_keywords` — matched against the headline (the tagline under the name).
- `certification_keywords` — e.g. `["AWS", "Google Cloud", "PMP"]`.
- `language_names` — full English language names the profile lists, e.g. `["English", "Spanish"]`.
- `school_names` — schools/universities attended, e.g. `["McGill University"]`.

**Company**
- `company_identifiers` — restrict to specific companies; each entry is a company domain (`amazon.com`), a LinkedIn company URL, or a Sales Navigator company URL/ID.
- `company_description_keywords` / `company_description_keywords_exclude` — keywords that must / must not appear in the company description.
- `company_sizes` — **controlled vocabulary** (head-count buckets), use only: `1`, `2-10`, `11-50`, `51-200`, `201-500`, `501-1,000`, `1,001-5,000`, `5,001-10,000`, `10,000+`.
- `company_industries_include` / `company_industries_exclude` — must match LinkedIn's standard industry taxonomy **exactly**, e.g. `"Software Development"`, `"Financial Services"`, `"Marketing Services"`, `"Hospitals and Health Care"`, `"IT Services and IT Consulting"`. A misspelled industry is ignored.

**Location**
- `country_names` / `exclude_country_names` — full English country names, e.g. `["United States", "United Kingdom", "Germany"]`.

**Numeric thresholds**
- `experience_count` — minimum number of distinct work experiences.
- `current_role_min_months_since_start_date` / `current_role_max_months_since_start_date` — tenure window in the current role (use the max to find recent joiners).
- `connection_count` — minimum LinkedIn connections.
- `follower_count` — minimum LinkedIn followers.

## Result shape

`search_profiles` returns `profiles_count` and `results[]`, where each item has:
- `linkedin_url` — pass straight into `modify_campaign_targets.items_to_add`.
- `full_name`, `job_title`, `company_name`, `location` — for plain-language display.

## Workflow

1. Turn the ICP from Milestone 0/1 into concrete filters (role + seniority + company size/industry + location + any keyword signals).
2. Run `search_profiles`. If you expect more than ~50 good matches, split into focused searches (per role, per region) rather than one broad query.
3. Present the matches to the user in **plain language** — name, title, company, location. Never surface raw field names, codes, or URLs as identifiers; let the user say which ones to keep.
4. Add the chosen `linkedin_url` values as targets via `modify_campaign_targets(request={ campaign_id, items_to_add: [...] })` (this happens in Milestone 2, after the campaign exists).

## Pushback

- "Just add everyone who matches" → narrow the filters; a diluted, oversized target list is expensive and lowers relevance (keep within the 10–50 sweet spot from `references/target-design.md`).
- A search returning 0 results usually means an over-exact controlled-vocabulary value (industry/size/seniority) — relax or correct it rather than piling on more filters.
