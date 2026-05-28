# Keyword Expansion

When the bottleneck is Target-bound on a ByKeyword campaign, the fix is "add more keywords" — but added wrong, this just floods the campaign with junk. Use these patterns.

## Expansion patterns

### 1. Adjacent topic

Pick an existing high-performing keyword. What does someone who writes about *that* also write about?

- "B2B SaaS pricing" → "B2B SaaS packaging", "enterprise sales discounting", "annual contract negotiation"
- "dbt incremental models" → "Snowflake query optimization", "data quality monitoring", "Airflow to Dagster migration"

### 2. Same audience, different pain

Same persona, different problem they post about.

- "scaling sales team" → "first sales hire", "sales onboarding playbook", "from founder-led to repeatable sales"

### 3. Same topic, different stage

Same topic from a different career/company stage.

- "PMF in fintech" (founder-stage) → "fintech regulatory expansion" (scale-up), "first 100 customers fintech" (very early)

### 4. Negative-anchored

If your topic has both interesting and noisy versions, write keywords that anchor toward the interesting one.

- Want senior eng posts, not bootcamp posts → use "principal engineer career", "staff engineer scope", "engineering ladder design"
- Avoid: "engineering jobs", "learn to code"

## What to avoid when expanding

- **Doubling a keyword you already have** with a synonym ("SaaS pricing" + "software pricing" + "subscription pricing") — these will match overlapping posts and inflate the search cost without adding reach.
- **Going one level too broad** ("AI", "data", "growth") — defeats the point of anchored expansion.
- **Trend-chasing terms** that decay fast ("agentic AI" was fresh 6 months ago, saturated now). Trends are fine but plan to rotate them every 3–6 weeks.

## How many to add at once

5–10 new keywords per expansion round. Less → not enough lift to measure. More → can't tell which ones worked when you audit next week.

## Validate after a week

After expansion, in the next audit:
- Did throughput rise? If yes, the new keywords are working — keep.
- Did ContentFilter rate rise? At least one new keyword is too broad — review and prune.
- Did ProfileFilter rate rise? At least one new keyword is hitting the wrong audience — review.

This is why the audit-grow-audit loop matters: keyword decisions are unverifiable without it.

## When to ask the user instead of brainstorming

If the campaign's domain is niche (specialized industry, narrow technical area, non-English market), the agent's general-knowledge brainstorm will produce mediocre keywords. Ask the user:

> "I can brainstorm generic expansions, but for {niche}, you'll know the vocabulary better than I do. What are 5 phrases people in your audience actually write in their posts? I'll vet them against the keyword design rules."

This is usually better than hallucinated keywords.
