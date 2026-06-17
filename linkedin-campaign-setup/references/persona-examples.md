# Persona examples (real filled-in campaigns)

Concrete models for filling the persona in Milestone 3 — drawn from real Liseller campaigns. Imitate the
**shape** and the **level of concreteness**, never the content. Complements `references/persona-design.md`
(what each field is for) and `references/ui-default-templates.md` (the blank UI templates). Still ask the
user for the real inputs — these only show you what "good" looks like.

## What good looks like (patterns per field)

- **product_description** — open with *Name + category* ("X is a [category] platform"), then the *core
  outcome* ("designed to turn Reddit into a measurable marketing channel"), then *how it works* in one or
  two concrete sentences. When the category is already well known, a sharp positioning angle ("the
  cost-effective Clay alternative") can stand in for a long description.
- **product_selling_points** — mechanism-level differentiators, never adjectives. Prefer *specific numbers*
  ($19/mo vs $495/mo, 24/7, "5-step waterfall = 5 actions"), *named comparisons* to a competitor for
  "alternative" positioning, and claims that are *provable*. "Runs in real browser sessions, no API
  shortcuts" beats "secure and reliable".
- **persona_description** — *role + tenure broken down by domain* + concrete expertise. Strong:
  "co-founder and tech leader — 5 years in data orchestration, 3 years in AI agents, 20+ years building
  startups". Weak (push back): "a professional and experienced leader with great expertise" — vague
  adjectives, no specifics. A name is optional; the years and domains are what carry it.
- **ai_brief** — start from the UI default template, then inject **1–3 campaign-specific imperatives**: a
  must-mention product framing, a required value style ("add value through numbers and facts"), or a
  must-include link. Keep the default structural bullets (relevance, optional author first name, concise,
  friendly tone, emoji only if the post uses one, newlines).
- **tone_and_style** — name the *register* ("professional yet enthusiastic"), the *balance* ("technical but
  optimistic"), the *format* ("conversational"), and any *signature markers* (specific emojis like 🚀).

## Worked example A — B2B SaaS, promo positioning

- **product_description:** "Feedheat is a Reddit-marketing automation platform that turns Reddit into a measurable marketing channel. It runs real browser sessions with AI agents that behave like genuine Reddit users — building reputation, engaging authentically, and promoting your product in relevant threads. The agents manage Reddit accounts, build karma, write posts, and join the right conversations 24/7."
- **product_selling_points:**
  - "Every agent runs a unique persona (occupation, interests, tone) so each post reads like a real recommendation, not an ad."
  - "Agents browse, upvote, and engage *before* any promotion — credibility first."
  - "Agents amplify each other and double down on what performs."
  - "Real browser sessions — no API shortcuts, so it looks like a real user to detection systems."
- **persona_description:** "Ana — Feedheat co-founder, tech leader and keynote speaker. 5 years in data orchestration, 3 years in AI agents, 20+ years building tech startups."
- **ai_brief (imperatives added on top of the default):** "Always frame Feedheat as a Reddit-marketing automation app; always weave in one or two of its key points most relevant to the post, varying which ones."
- **tone_and_style:** "Professional yet enthusiastic; technical but optimistic about AI; conversational; an occasional 🚀."

## Worked example B — "cost-effective alternative", number/comparison-led

- **product_description:** "Latenode is a workflow-automation platform — the cost-effective Clay alternative."
- **product_selling_points:** "Clay $495/mo vs Latenode $19/mo. Clay bills two resources (data credits + actions); Latenode charges by execution time, so a 5-node workflow costs the same as a 1-node one at equal runtime. Free templates to migrate from Clay."
- **persona_description:** "Ana Ant — startup leader with hands-on expertise in AI and automation tooling." *(Still thin — push for years + a specific shipped win before using it.)*
- **ai_brief (imperatives added):** "Be helpful; add value through numbers and facts; always include the migration link <url>."
- **tone_and_style:** inherit the professional-yet-enthusiastic default.

## Tensions to handle (don't copy blindly)

- **Promo intensity:** a "always mention <product>" imperative suits a **promo** or **mixed** campaign. On
  a **neutral** campaign, drop it — forced product mentions read as spam. Match the ai_brief's promo level
  to the chosen `commenting_mode`.
- **Always-include-a-link:** a per-comment link (Example B) is fine **only if the user explicitly asks** —
  it looks promotional and can dent engagement. If you put it in `ai_brief`, do **not** also list "no
  links" in `avoid`, and tell the user the trade-off.
