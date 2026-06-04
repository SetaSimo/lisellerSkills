# Target Design Guide

The single biggest predictor of campaign success is target quality. Spend 80% of setup time here.

## ByKeyword: writing good keywords

### The ICP framing

A good keyword is one that surfaces posts the user's **ICP** (Ideal Customer Profile) would naturally **read, search for, or write themselves**. Not topics adjacent to the product. Not branded terms. The posts that *would already be in the ICP's feed*.

Practical test before adding a keyword: "If a member of the ICP saw this post in their feed, would they stop scrolling?" If no — drop it.

### Rules

1. **1–3 words, never more.** Each keyword is at most three words. Brand names ("Kajabi"), tight acronyms ("GTM", "revops"), and 2–3-word phrases ("business coaching", "client acquisition coaching", "revenue operations") all work. Sentences and 4+ word phrases fragment LinkedIn search and get filtered out.

2. **Anchor to audience or pain.** A keyword should map to *something the user's audience writes about*. "client retention" anchors the topic; "sales" anchors nothing.

3. **Avoid over-promoted terms.** Hashtag-style terms ("#AI", "#crypto") attract spam and ghost engagement. Prefer natural-language phrases that show up in actual posts.

4. **Mix specific and aspirational.** Pure specifics ("Snowflake cost") get low volume on their own. Pure aspirationals ("future of work") get noise. A campaign of ~30 keywords should balance both.

5. **Negative space matters.** If a topic has a common but irrelevant meaning ("python" → snake), pick a tighter alternative or skip it.

6. **Use language the ICP uses.** If they call it "RevOps", "revenue operations" misses. If they say "GTM", "go-to-market strategy" misses. Mirror their vocabulary, not marketing copy.

### Examples

**Goal: "be seen by online coaches and course creators"**

Good keyword set (1–3 words, in the language coaches actually use):
- "business coaching"
- "coaching clients"
- "coaching industry"
- "coaching offers"
- "course creators"
- "Kajabi"
- "online coaching"
- "personal development"
- "Tony Robbins"
- "executive coaching"
- "coaching funnel"
- "coaching business"
- "life coaching"
- "high ticket coaching"

**Goal: "be seen by agency owners and digital marketing agencies"**

Good keyword set:
- "agency clients"
- "agency owner"
- "client growth"
- "client retention"
- "digital marketing"
- "marketing agency"
- "marketing funnels"
- "social media growth"
- "agency revenue"
- "agency scaling"
- "lead generation"
- "agency growth"
- "scaling agency"
- "agency retention"

**Goal: "be seen by RevOps and GTM leaders"**

Good keyword set:
- "revenue operations"
- "Go-to-market"
- "revops"
- "GTM"

### How many?

**Target ~30 keywords (aim for 25–30).** Generate the full set yourself from the ICP and the user's profile/company context, then present them for the user to trim or extend. Fewer than ~20 risks the campaign idling; keep every keyword relevant (1–3 words, audience-anchored) so the content filter doesn't eat the budget on noise.

**Hard ceiling: ≤ 500 keywords per campaign.** Beyond that, split into separate campaigns.

**Cost scales linearly with target count.** Every keyword is run as its own search once per round, and the per-search price is billed for each. Doubling the keyword list doubles the search spend regardless of how many posts come back — so keep the ~30 tight and relevant rather than padding toward the ceiling.

## ByProfile: building a good profile list

### Rules

1. **All URLs must match** `https://[www.]linkedin.com/{in|company|showcase}/<slug>` (optional trailing slash and query string). The MCP will reject invalid URLs.

2. **Verify the person/company is *actively posting*.** A target that posts twice a year is a dead slot. If you can, ask the user to confirm each profile has posted in the last 30 days.

3. **Mix tiers.** 100% A-list executives → low post frequency, high competition for comment visibility. 100% mid-tier → low engagement returns. Aim for ~30% high-profile, ~50% mid-tier active posters, ~20% peers (which builds genuine community).

4. **10–50 profiles is the relevance sweet spot** (hard ceiling ≤ 500 per campaign). Under 10 → campaign starves between posting cycles. Over 50 → attention dilutes and per-profile limits gate everything. Each profile URL is one search per round at `prices.search`, and ByProfile additionally forces a profile parse + `prices.profile_filter` per evaluated post — so per-target cost is higher than ByKeyword and still scales linearly with the list size.

### Example

**Goal: "ABM into 20 target accounts at fintech companies"**

For each target account, the user should provide:
- 1 company page URL (so the campaign sees company posts)
- 2–3 individual decision-maker URLs (CTO, VP Eng, Head of Data)

That's 60–80 URLs for 20 accounts — manageable, focused, and surfaces multiple post streams per account.

### Discovering profiles with `search_profiles`

If the user wants you to **find** profiles rather than supplying their own list, use `search_profiles`
to discover candidates by criteria. **Only when the user explicitly asks to search** — never to guess
a known person's URL, and never for ByKeyword campaigns.

It returns up to **50** matches per call, each with a profile URL ready to add as a target. You can
filter by job title, seniority, company (name / domain / industry / size), location, languages, bio
keywords, certifications, schools, and connection / follower counts — the tool's own parameters carry
the exact accepted values, so build the call from those rather than from a list memorised here.

Workflow: turn the ICP into filters → search → present the matches to the user **in plain language**
(name, title, company, location — never field names or codes) and let them pick → add the chosen URLs
as targets. The 50-row cap is per call; narrow the filters or run a few focused searches rather than
asking for "everyone".

## Mistakes to push back on

- **"Add my whole CRM as profile targets" (500+ URLs)** → diluted, slow, expensive. Suggest splitting into segments.
- **"Use the company name as a keyword"** → not how LinkedIn search works; this will match unrelated posts that mention the name in passing. Use ByProfile with the company page URL instead.
- **"Just use one keyword: my product name"** → almost no one posts your product name organically. Use audience-anchored keywords instead.
- **"Copy keywords from my Google Ads"** → search-intent keywords are different from post-content keywords. SEM intent ≠ what people post about.
