# Milestone 5 — Dry-run review (+ cost & runway)

Read this after Milestone 4. It covers the original Step 12. When you finish it, return to the
milestone map in `SKILL.md` and open the Milestone 6 file (complete + launch). Run the
`references/setup-checklist.md` pass before you present the review.

## Step 12 — Dry-run review

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

1. `get_user_stats` → balance + `prices`; `list_campaigns(areDetailsRequired=true)` → for **every running campaign plus this new one**, estimate daily spend and sum into one **combined daily spend**. Cost formula in `references/limits-and-methods.md`.
2. Runway: the monthly plan budget refills on its renewal date; the top-up balance covers spend once the monthly budget is gone. Tell the user, in plain prose and as a range: combined daily spend across N running campaigns, that the monthly budget lasts until ≈ its renewal date, and ≈ how many total days the balance keeps everything running. Say "your monthly plan budget" / "top-up balance" — never raw numbers or field names.

**Wait for explicit user approval before proceeding.**

---

**Milestone 5 done.** Return to the milestone map and open `references/milestone-6-complete-and-launch.md`.
