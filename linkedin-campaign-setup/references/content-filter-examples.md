# Content Filter Instruction Examples

Real-world content-filter instructions from production users. These are the patterns that actually work — the model is sensitive to format.

**Two hard rules for every instruction:**

1. **First line is mandatory** — always `I am looking for posts about <broad topic that unites the user's keywords>` (or `I am looking for posts from <broad ICP description>`). Derive the broad topic from the user's keyword list: e.g. keywords about coaching → "coaching, online courses, and personal development"; agency-marketing keywords → "agency growth, client acquisition, and digital marketing"; RevOps/GTM → "revenue operations and go-to-market strategy". This opening line anchors the AI back to the topic when many skip rules follow.
2. **Only objective skip criteria** — every skip rule names something measurable: content shape ("only hashtags", "only a link", "only emoji"), a literal brand/word, an author role/title. **Never** subjective rules like "nothing useful to add", "posts the AI can't engage with", "low-quality posts" — those produce inconsistent behaviour and can't be audited.

The two default rules from Step 6 (reject link-only / reject emoji-only) are appended at the bottom of every instruction. The blocks below are the **user-specific** part.

---

## Pattern 1 — Simple ICP one-liner

For tightly-defined audiences where a single sentence is enough:

> `I am looking for posts from agency owners or digital marketing professionals managing or growing client portfolios.`

When to use: clear ICP, no edge cases, the AI doesn't need disambiguation. Combine with the 2 default rules and you're done.

---

## Pattern 2 — Topic + flat objective skip list

For wide ICPs that need to be focused via explicit, objective exclusions. The instruction stays short — one topic line, then a flat list of `Skip posts that …` / `Skip posts whose author …` rules. **No "Prioritise" lines** — every rule is binary skip-or-keep.

> ```
> I am looking for posts about land subdivision, property development, and civil engineering in Australia.
> - Skip posts that contain only hashtags.
> - Skip posts that contain only a link or only a reshared post with no original text.
> - Skip posts that mention "HIRING", "Career", "job", "recruitment", or "intern".
> - Skip posts whose author is a recruiter, student, or job seeker.
> - Skip posts whose author is located outside QLD, NSW, or VIC.
> ```

Structure to copy:
1. **Opening line** — `I am looking for posts about <broad topic that unites the keywords>` (one sentence, no bullet list inside).
2. **Flat list** — each rule is one bullet starting with `- Skip posts that …` or `- Skip posts whose author …`.
3. **Objective only** — every rule references something measurable: a literal string ("HIRING", "Proofpoint"), a content shape (hashtags-only, link-only, emoji-only), a length cutoff, or a role/title that appears in the headline.

If the user asks "what about posts I should prioritise?" — explain that the filter has no priority concept; it's a binary keep-or-skip gate. Anything not skipped is kept. To narrow harder, add more `Skip` rules.

---

## Pattern 3 — Technical niche with brand blacklist + hiring filter

For competitive verticals where users want to avoid commenting under competitor posts or job listings:

> ```
> I am looking for posts about cybersecurity.
> - Skip post if there are only hashtags in it.
> - skip posts if there is only words of Hiring, job, jobs, recruitment, hire.
> - Skip posts if it mentions "Proofpoint", "Mimecast", "Abnormal", "KnowBe4",
> - skip posts that mention "HIRING" or " contract" or "Career" or "job" or "career"
> - skip posts that mention "SlashNext"
> ```

Structure:
- **Opening line** — `I am looking for posts about <topic>.`
- **Skip block** — flat list, each rule on its own line starting with `- Skip` or `- skip` (mix OK, model handles both).
- **Brand names in quotes** — `"Proofpoint"` matches literally; without quotes the model gets fuzzy. Always quote brand strings.
- Hiring keywords — list ALL CAPS variants too (`"HIRING"`, `"Career"`) — LinkedIn job posts disproportionately use uppercase.

---

## Pattern 4 — Minimal one-liner + anti-spam

When the ICP is narrow and only one anti-spam rule is needed:

> ```
> I am looking for posts discussing seamless, proactive email security solutions.
> - Skip post if there are only hashtags in it.
> ```

The duplicate-with-defaults rule is OK — the user wrote it explicitly so it stays in their copy. Don't strip it.

---

## Format rules the model is sensitive to

- **Opening line is mandatory.** Every instruction starts with `I am looking for posts about <broad topic>` or `I am looking for posts from <broad ICP description>`. The model has been observed to perform better with this opener vs alternatives like "Find posts ..." or "Accept posts ...". Don't change the verb.
- **`Skip posts that …` / `Skip posts whose author …` / `Skip post if …`** — all work. Stick to one phrasing within a single instruction for consistency.
- **Objective criteria only** — content shape (hashtags-only, link-only, emoji-only), a literal string ("HIRING", "Proofpoint"), an explicit role/title in the headline. **No subjective skips** ("nothing useful to add", "low-quality posts"); the filter cannot evaluate them reliably.
- **No "Prioritise" lines.** The filter has no priority concept; it's a binary keep-or-skip gate. Anything not skipped is kept. To narrow harder, add more `Skip` rules.
- **Quote brand names and exact strings** to match: `"Proofpoint"`, `"HIRING"`. Unquoted multi-word skip phrases will fuzzy-match too broadly.
- **Geo clauses** — say countries/states explicitly (`QLD`, `NSW`, `VIC`) in the content-filter text rather than relying on a profile-level geo list alone: the content filter reads the post text where the author often states the regions they operate in, and the profile-level geo filter additionally forces a profile parse per post and is billed per evaluated post. Only set a profile-level geo list when LinkedIn-profile geo scoping is genuinely required on top of the text rule.
- **The 2 default rules are appended automatically** by the setup skill (Step 6) — don't repeat them in your block unless the user explicitly wrote them.

---

## Sequencing

When applying, combine the user-specific block (one of the patterns above) with the 2 default rules at the end:

```
I am looking for posts about <broad topic that unites the user's keywords>.
<user-specific Skip rules>

Reject posts that contain only a link with no original text.
Reject posts that contain only emoji with no substantive text.
```

Send the combined instruction as a single string in the assistant content-filter patch.
