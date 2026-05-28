# UI Default Templates for Persona Fields

Every persona field in the web UI ships with a default template containing `{{...}}` placeholders. Users have seen these defaults and are familiar with their structure — when the setup skill needs to fill a persona field, **use these templates as the baseline** and substitute the placeholders with what the user told you. Do not invent your own shape.

**Hard rule:** never submit a field via `update_campaign_assistant.persona_patch` while it still contains unsubstituted `{{...}}` placeholders. Read the substituted text back to the user and get sign-off first.

---

## `product_description`

```
{{Company Name}} is a versatile platform designed to simplify {{core function or service}}. It offers {{Key features}} to support the needs of {{target audience or industry}}. With a focus on {{primary benefits, such as scalability, ease of use, or flexibility}}, it delivers solutions that enhance {{specific outcomes or tasks}} for a variety of users.
```

Placeholders to fill from the user's Step 1 answers:
- `{{Company Name}}` — the product or company being promoted
- `{{core function or service}}` — one-line "what does it do"
- `{{Key features}}` — 2–4 specific features the user named
- `{{target audience or industry}}` — who buys it (ICP)
- `{{primary benefits, ...}}` — pick from scalability / ease of use / flexibility / cost-savings / etc.
- `{{specific outcomes or tasks}}` — the job-to-be-done

---

## `product_selling_points`

```
- High-quality product or service: Our products/services are designed with a focus on quality, ensuring long-lasting performance and customer satisfaction.

- Cost-effective solutions: We offer competitive pricing without compromising on value, making our solutions affordable for businesses of all sizes.

- Exceptional customer support: Our team is dedicated to providing responsive and personalized support to ensure you get the most out of our offerings.

- Innovation and technology: We leverage the latest technology to deliver cutting-edge solutions that drive efficiency and growth.

- Scalability and flexibility: Our solutions grow with your business, offering the flexibility to adapt to changing needs and demands.
```

Each bullet is a category. Ask the user to **replace generic phrasing with their concrete differentiator** for each category — keep the bullet structure, swap the body for what's actually true. Use `\n` between bullets when sending the patch.

---

## `persona_description`

The UI ships the template below, but **the name is not required** — the AI references the persona to ground comments in expertise, not to sign them. **What actually matters is the persona's background: roles held, years of experience in the relevant field, technical/domain strengths, what they've shipped or led, and the topics they care about.** Collect those; the name is optional flavour.

```
{{Persona Name (optional)}} is the {{Job Title}} of {{Company Name}}, a {{brief description of the company, e.g., a no-code automation platform}}.
With {{number of years}} of experience in {{relevant field}}, {{they}} have helped shape {{Company Name}} to address {{specific industry challenges or product differentiators}}.
Before joining {{Company Name}}, {{they}} worked on {{highlight relevant professional experiences or achievements}}. Strengths: {{2-4 concrete strengths — e.g. "scaling distributed systems", "B2B pricing", "infra cost optimisation"}}. Passionate about {{specific topics or skills}}; committed to {{goal or mission relevant to the company}}.
```

If the user doesn't want to provide a name, drop the `{{Persona Name (optional)}}` token entirely and start the description with "The author is the {{Job Title}} of {{Company Name}}..." or similar — the AI generates comments without ever printing a name.

**Required fields to extract from the user** (substitute these into the template):
- Job title / role
- Years of experience in the relevant field
- 2–4 concrete strengths or areas of expertise (specific — "Postgres performance", not "engineering")
- Notable prior experience / what they've shipped
- Topics they care about (drives which posts they engage substantively with)

**Optional:** name (only if the user wants it on record — it does NOT appear in published comments).

---

## `ai_brief` (UI label: "What AI need to do") — REQUIRED

```
Given a LinkedIn post, generate a concise and engaging comment from my perspective ({{Your persona}} described below). The comment should:

- Be relevant to the content of the post and reflect genuine interest.
- Sometimes include the first name of the post author, if provided in the "Name" field and if it appears to be a real person's name. If the "Name" field is blank or doesn't resemble a real name, omit the use of a first name.
- Add value to the conversation through insights or constructive feedback.
- ALWAYS mention one or two of {{Company Name}}'s Key points that are most relevant to the post and fit the context (listed below). Avoid consistently using the same points.
- Keep the comment concise, conversational, and easy to read without overwhelming details.
- Maintain a professional and friendly tone.
- Include ONE natural-looking emoji only if the original post uses emojis.
- Use new lines to improve readability of the comment.
```

Placeholders:
- `{{Your persona}}` — a one-line summary of `persona_description` (e.g. "a Head of Sales Ops at a Series B SaaS company")
- `{{Company Name}}` — same value used in `product_description`

The campaign **will not start** while `ai_brief` is empty or whitespace — `set_campaign_running(enabled=true)` performs a pre-flight check.

---

## `avoid` — UI default Do-Not list

```
- Don't be overly promotional or salesy.
- Don't write overly long or unnatural comments.
- Don't add sign to message
- Don't add hashtags in comments.
- Don't use these phrases: "game changer"
```

This is the user-facing baseline. Ask the user to add their own banned phrases or rules (e.g. "no emojis", "never compare to named competitors", "don't use the word 'leverage'"). **Keep `avoid` to persona-level rules only** — content-filtering rules (skip posts of kind X, skip non-English) belong in `content_filter_patch.instruction` (see Step 6 + `content-filter-examples.md`).

---

## `tone_and_style`

```
The tone is professional yet enthusiastic, aiming to inspire confidence in technological progress. It balances a knowledgeable, technical approach with an optimistic outlook on innovation. The style is conversational, engaging the reader with a positive message about the benefits of AI-driven solutions. The use of upbeat phrases and emojis, like 🚀, gives it a modern, forward-thinking vibe while still maintaining a formal, expert tone.
```

This default leans tech-optimistic. For users whose persona is more cynical / B2B-dry / academic / etc., **rewrite this section** rather than tweak — read the substituted version back before submitting.

---

## Workflow

1. Collect user answers in Step 1.
2. Open each template above, substitute every `{{...}}` from the collected answers.
3. **Show the final text** to the user for each field and ask «is this accurate?»
4. Only after sign-off, send via `update_campaign_assistant(campaign_id, patch={ persona_patch: { product_description, product_selling_points, persona_description, ai_brief, avoid, tone_and_style, is_comment_generation_enabled: true } })`.
5. If the user says «yes, this is fine» for the template body — fine, but you still must have substituted every `{{...}}`. The campaign will start without complaint even if `ai_brief` contains literal `{{...}}` (the gate only checks non-empty), but the AI output will be garbage. Never ship unsubstituted placeholders.
