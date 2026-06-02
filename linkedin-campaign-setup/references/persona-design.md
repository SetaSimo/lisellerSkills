# Persona, Product, and Commenting Design

Step 8 of the setup walks the user through the AI's voice. These fields are stored on `DOpenaiAssistant` and consumed every time the AI writes a comment — bad inputs produce bad comments at scale.

**Hard rule: don't fabricate.** These fields must come from the user. If they can't articulate a field, summarize what they told you in Step 1, read it back, and ask "is this accurate?" before writing the patch.

## What goes where: persona vs content filter vs profile filter

Some rules look like persona rules but actually belong in the **filters**. The distinction:

- **Persona / `avoid`**: rules about *how* the AI writes when it does write. Brand-voice rules, banned phrases, tone constraints. Applied at comment-generation time.
- **Content filter** (`content_filter_patch.instruction`): rules about *which posts* are worth engaging with **and which authors to engage**. The instruction is matched, in **one** request, against the **post text** (reshared + original) AND the author's **LinkedIn Headline** (captured at search time — no profile parse needed) plus the language whitelist. Charged `prices.content_filter` per evaluated post; runs once per post and short-circuits comment generation when rejected.
- **Profile filter** (`profile_filter_patch`): geo only — the author's geo against `geo_whitelist` / `geo_blacklist`. Enabling it forces a profile parse for every candidate post and is billed per evaluated post (`prices.profile_filter`) — see Step 6 of the setup skill. The `filter_open_to_work` / `filter_hiring` switches are accepted but **not enforced by the live pipeline today** — do not rely on them.
- **Key consequence:** author / role / Headline rules ("only CTOs", "skip students", "skip recruiters") go in `content_filter_patch.instruction` — the content filter applies them to the author Headline in the same paid content-filter call, with **no** profile filter, **no** profile parse, and **no** `prices.profile_filter` cost. The profile filter is only for geo. There is no separate author-headline field, and `filter_open_to_work`/`filter_hiring` do nothing.

Mental model: content filter = "is this post worth a comment AND is this author the right kind?" (post text + author headline, one paid call). Profile filter = "is this author in an allowed geo?" (geo only, separate paid call, forces a profile parse). Persona/avoid = "given that we're commenting, how should it sound?".

Examples — put these in the **content filter**, NOT in `avoid`:

- "No comments in languages other than English" → `language_whitelist=["English"]`
- "Don't comment if you have nothing specific to add — better silent than generic" → content filter instruction
- "Skip posts that are only links / only emoji" → content filter instruction (Step 6 default)
- "Skip posts about hiring / mental health / politics" → content filter instruction
- "Skip students / pre-career profiles" (an **author/role** rule) → write it into `content_filter_patch.instruction` (e.g. "Skip if the author is a student, intern, or otherwise pre-career"). The content filter matches it against the author Headline automatically — no profile filter, no parse, no extra cost. Do NOT use `filter_open_to_work` — it is not enforced.

Examples that belong in **`avoid`**:

- "No self-promotion, no links, no DM me, no check my profile"
- "No generic praise (great post!, love this!, 100% agree)"
- "Never compare us directly to named competitors"
- "Don't use the word leverage / synergy"
- "No emojis"
- "Don't pretend to have experience you didn't describe in persona"

If the user produces an `avoid` list that mixes both kinds, split it: keep persona-level rules in `avoid`, move post-selection rules into the content filter instruction in Step 6.

## Persona patch fields

Submitted via `update_campaign_assistant.persona_patch`. Each text field is a full replacement (not append). All are optional individually — only send the ones the user has answered.

### `product_description`
- 1–3 paragraphs.
- What it is, what category it sits in, who it's for.
- **Bad:** "The best CRM ever."
- **Good:** "A CRM specialized for B2B SaaS sales teams of 5–50 reps. Replaces Salesforce for companies that find it too heavy."

### `product_selling_points`
- Bullet list, one per line (use `\n`).
- Concrete differentiators, not adjectives.
- **Bad:** "Easy to use, powerful, scalable"
- **Good:**
  ```
  - 10-minute setup vs 6-week Salesforce rollout
  - Built-in revenue forecasting, no separate BI tool
  - Flat $50/seat — no per-feature add-ons
  ```

### `persona_description`
- **What the AI actually uses:** role, years of experience, concrete strengths/expertise, prior companies/achievements, and topics they care about. These become the grounding facts the AI leans on when forming an opinion in each comment.
- **Names are NOT required** — the AI never signs comments with the persona's name. Don't insist on it. If the user offers one, fine; if not, drop the placeholder entirely and the persona description starts with "The author is..." or just "is the {{Job Title}}...".
- **Bad:** "A friendly expert"
- **Good (no name):** "Head of Sales Ops at a Series B SaaS company. 12 years in B2B sales tooling. Strengths: pipeline forecasting, CRM rollouts, RevOps tooling selection. Previously ran ops at two early-stage SaaS companies. Occasionally cynical about CRM bloat but constructive."
- **Also good (with optional name):** "Alex Chen, Head of Sales Ops at a Series B SaaS company. 12 years in B2B sales tooling..."

### `avoid`
- Do's & Don'ts. Persona-level only (see split above).
- **UI default** (use as baseline, then ask the user for additions):
  ```
  - Don't be overly promotional or salesy.
  - Don't write overly long or unnatural comments.
  - Don't add sign to message
  - Don't add hashtags in comments.
  - Don't use these phrases: "game changer"
  ```
- Common user-specific additions (offer when relevant):
  - No self-promotion, no links, no "DM me", no "check my profile"
  - No generic praise ("great post!", "love this!", "100% agree")
  - Never compare us directly to named competitors
  - Don't use the word "synergy" or "leverage"
  - Don't pretend to have experience you didn't describe in persona

See `references/ui-default-templates.md` for the full UI-default `avoid` template.

### `tone_and_style`
- Register, formality, length preference.
- **Bad:** "Professional but friendly"
- **Good:** "Conversational, sentence case, 2–4 sentences max. Skip greeting lines. Punchy openers. Use first-person singular."

### `ai_brief` ("What AI need to do") — REQUIRED

- Top-level instructions injected before everything else.
- **Mandatory before starting the campaign**: `set_campaign_running(enabled=true)` rejects the request while this field is empty or whitespace on the linked assistant. Empty `ai_brief` is the most common reason a campaign refuses to start, after missing targets.
- This is the "what should the AI actually do, in your own words" field — not a place for filtering rules (those go in `content_filter` / `profile_filter`) and not a place for voice rules (those go in `tone_and_style` / `avoid`). It's the *job description* given to the AI per post.
- Capture it from the user. If the user can't articulate one, propose the default template below, fill the placeholders from the rest of the persona, and read it back for sign-off before submitting:
  ```
  Given a LinkedIn post, generate a concise and engaging comment from my perspective ({{persona_description}} described below). The comment should:

  - Be relevant to the content of the post and reflect genuine interest.
  - Sometimes include the first name of the post author, if provided in the "Name" field and if it appears to be a real person's name. If the "Name" field is blank or doesn't resemble a real name, omit the use of a first name.
  - Add value to the conversation through insights or constructive feedback.
  - ALWAYS mention one or two of {{product_name}}'s Key points that are most relevant to the post and fit the context (listed below). Avoid consistently using the same points.
  - Keep the comment concise, conversational, and easy to read without overwhelming details.
  - Maintain a professional and friendly tone.
  - Include ONE natural-looking emoji only if the original post uses emojis.
  - Use new lines to improve readability of the comment.
  ```
- Never ship the literal `{{...}}` placeholders. Replace them with the user's `persona_description` summary and the product/company name from `product_description` before sending the patch.

### `is_comment_generation_enabled`
- Set to `true` once the persona fields are filled. The default is `false` so the AI can't ship without a voice.

## Commenting patch — pick exactly one mode

Submitted via `update_campaign_assistant.commenting_patch.commenting_mode` as a single string. The MCP layer translates this into the underlying flags ensuring exactly one is set.

| Mode | Set true | When |
|---|---|---|
| Promo only | `commenting_mode: "promo"` | Every comment pushes the product. Highest CTR, lowest community trust. Use when the audience already knows you and visibility on the product is the goal. |
| Neutral only | `commenting_mode: "neutral"` | Every comment is value-only — share an opinion, ask a question, add a stat. Lowest CTR, highest community trust. Use when the goal is to *be seen as a thoughtful voice in the niche* before the product comes up. |
| Mixed | `commenting_mode: "mixed"` | The AI chooses per-post. Balanced. **Recommended default** for most new campaigns. |

When you switch modes on an existing campaign, just send the new `commenting_mode` value — the MCP layer rewrites the three underlying flags for you.

**Don't pre-fill the mode** — ask the user explicitly and present the three options with pros/cons. If they say "you choose", recommend `"mixed"` and explain why. Full ask-script:

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

## Likes

There's no separate "assistant may like" toggle anymore. Whether the campaign publishes likes is controlled solely by the campaign-level `auto_like_enabled` (Step 11).

## `opener_lines` (Pro only)

- Server rejects a non-empty `opener_lines` value unless the AI method is Pro (code `premium`).
- Free-form opener bank. The Pro AI samples from these to start its comment. **Pro also performs background research** (fresh news, analytics on the post topic) before writing — these openers prime the research-backed comment with a natural-sounding lead-in so it doesn't read like a wikipedia citation.
- Recommended: 5–15 distinct openers, one per line. **Don't ship a single opener** — the AI will recycle it and the campaign will look bot-like.

Reference openers the user has approved as a stylistic baseline:

```
I was just reading about this, interestingly...
I recently came across research showing...
I've been exploring this lately, and...
Saw some data on this last week...
This reminds me of what...
```

Use these as a starter set and ask the user to add their own in the same register. Submit the combined list as `opener_lines` separated by `\n`.

- Pass empty string to clear an existing value (e.g. if the user explicitly switches to Common — clear first, then change the method).

## Cross-check before submitting

- Have you collected concrete examples for `product_selling_points` and `avoid` rather than vague adjectives?
- Does `persona_description` give the AI enough credibility context that it won't sound like a marketing intern?
- Does `tone_and_style` constrain length and register, or is it generic ("professional")?
- Are post-selection rules ("skip X kind of post", "no language Y") in the **content filter**, not in `avoid`?
- Since the method is Pro by default, did you collect at least 5 `opener_lines`?
- Is exactly one `make_*` mode `true`?

If any of these is no, go back to the user. Bad personas produce months of bad output — get explicit sign-off.
