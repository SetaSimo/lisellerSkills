# Milestone 3 — AI method + persona + comment type

Read this after Milestone 2. It covers the original Steps 7–8. When you finish it, return to the
milestone map in `SKILL.md` and open the Milestone 4 file.

## Step 7 — Choose AI method

`update_campaign(campaign_id, patch={ ai_assistant_method_id })` with the id from `list_lookup(kind: "AiMethod")`.

**Default to Pro (code `premium`) and set it now** for every campaign — it performs background research and writes evidence-backed comments (multiple-times-higher engagement) and unlocks `opener_lines`. Use `Common` only on explicit request; `Off` only for intentional likes-only; **never** `Custom` (UI-only). You may state Pro's value qualitatively, but quote no price here. Full method table in `references/limits-and-methods.md`.

## Step 8 — Persona + comment type + (Pro) opener lines

Single call: `update_campaign_assistant(campaign_id, patch={ persona_patch, commenting_patch })`.

**Persona (`persona_patch`)** — every text field is a full replacement (not append). Don't fabricate these from thin air — ask the user. **The UI ships with default templates with `{{...}}` placeholders for every persona field; use those as the baseline rather than inventing new shapes.** See `references/ui-default-templates.md` for the verbatim UI defaults, `references/persona-design.md` for what each field is for, and `references/persona-examples.md` for compressed real-world fills to model the level of concreteness.

Fields:
- `product_description` — what is being promoted (1–3 paragraphs). UI default template starts with `{{Company Name}} is a versatile platform designed to simplify {{core function or service}}...`
- `product_selling_points` — bullet list (use `\n` between items). UI default: 5 bullets (High-quality / Cost-effective / Exceptional support / Innovation / Scalability).
- `persona_description` — who is supposed to be writing the comments. **A name is NOT required** — what matters is the **role, years of experience, concrete strengths, prior achievements, and topics of interest**. The AI never signs comments with the persona's name; it uses the persona as background context to ground its takes. UI default starts with `{{Persona Name (optional)}} is the {{Job Title}} of {{Company Name}}...` — drop the name token if the user prefers, see `references/ui-default-templates.md`.
- `avoid` — Do's & Don'ts. Keep these to **persona-level** rules (e.g. "no emojis", "never compare to competitors by name", "don't use the word leverage"). **Don't** put rules like "no comments in languages other than English" or "skip posts with no substance" here — those belong in the **content filter** (Milestone 2) where they apply once per post and are auditable. UI default: 5 bullets (no overly-promotional / no overly-long / no sign / no hashtags / banned phrases).
- `tone_and_style` — register / stylistic guidance. UI default starts with `The tone is professional yet enthusiastic...`.
- `ai_brief` — **REQUIRED** (UI calls this "What AI need to do"; JSON name is `ai_brief`, underlying C# field is still `BasicInstructions`). The server rejects `set_campaign_running(enabled=true)` **and** `complete_campaign_setup` while it is empty or whitespace. Capture the user's own description of how the AI should approach each post. If the user has nothing specific, use the UI default template (with `{{Your persona}}` / `{{Company Name}}` placeholders filled in from the rest of the persona) — see `references/ui-default-templates.md`.
- `is_comment_generation_enabled=true` once the persona is filled

**Hard rule:** never send a field whose value still contains unsubstituted `{{...}}` placeholders. Always read the final text back to the user and ask for sign-off before the `update_campaign_assistant` call.

**Comment type (`commenting_patch.commenting_mode`)** — **not** a silent default. Ask the user explicitly which of `"promo"` / `"neutral"` / `"mixed"` they want, present the pros/cons, then submit `commenting_mode: "<choice>"`. If they say "you choose", recommend `"mixed"`. Full three-mode ask-script with pros/cons in `references/persona-design.md`.

**`opener_lines` (Pro only)** — send 5–15 distinct openers (one per line) for the Pro AI to sample; use the reference set in `references/persona-design.md` as a baseline and ask the user to add their own. **Never send `opener_lines` when the method isn't Pro — the server rejects the patch.**

---

**Milestone 3 done.** Return to the milestone map and open `references/milestone-4-cadence-limits-name-toggles.md`.
