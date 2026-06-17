# Milestone 6 — Complete setup, launch, follow-up

Read this after Milestone 5, once the user has approved the dry-run. It covers the original Steps 13–14,
rewired to the dedicated completion tool. This is the **last** milestone — but a campaign is not "done"
until it is either launched or the user has explicitly declined to launch it.

## Step 13 — Complete setup, then start

Two calls, in order:

1. **`complete_campaign_setup(campaign_id)`** — the dedicated validate-and-complete tool. It checks everything required for a campaign to run and reports back a strongly-typed result:
   - `is_setup_completed: true` + empty `missing[]` → setup is complete, you may start the campaign.
   - `is_setup_completed: false` + a `missing[]` list → **nothing was changed.** Each entry has a `code` and a plain-language `message` (a filled persona ai_brief, at least one target, and — when `is_using_schedule=true` — at least one schedule window). It reports **all** problems at once: fix every item (in plain language to the user, never raw codes), then call `complete_campaign_setup` again. The call is idempotent once completed.

   This is the **only** way to mark a campaign setup-complete — `update_campaign` no longer sets `is_setup_completed`.

2. **`set_campaign_running(campaign_id, enabled=true)`** — dedicated start/stop tool. The server runs four pre-flight checks and rejects the call if any of them fails:
   - `is_setup_completed != true`
   - the assistant's `persona.ai_brief` ("What AI need to do") is empty or whitespace
   - the campaign has zero targets (keywords for ByKeyword, profile URLs for ByProfile — added via `modify_campaign_targets`)
   - `is_using_schedule=true` but no schedule windows are configured

   On success the server also recomputes the next-run timestamp: a random time inside the next scheduled window (when `is_using_schedule=true`) or `now + minimal_round_interval` (otherwise). Pair with `set_campaign_running(..., enabled=false)` to stop. `update_campaign` cannot flip `is_enabled` — use `set_campaign_running`.

### Always offer to launch — even mid-next-campaign (do not skip)

After `complete_campaign_setup` succeeds you **must** offer to start the campaign, and you must not let a finished-but-unlaunched campaign quietly fall through the cracks:

- If the user approved at the dry-run and wants it live, call `set_campaign_running(enabled=true)` and confirm in plain language ("Your *Fintech founders ABM* campaign is now live and will start looking for posts within the hour").
- If the user has **already moved on to creating another campaign** before launching this one, do not silently continue. Explicitly ask whether to start the just-finished one first — e.g. «Before we build the next one — do you want me to switch on *Fintech founders ABM* now, or leave it paused for you to start later?» Then mark its launch status in your session ledger accordingly.
- Update the **session ledger** (see `SKILL.md`): record each campaign you finish as *launched* or *awaiting launch*. Whenever you wrap up a campaign — or the user says they're done creating campaigns — re-surface any campaign still *awaiting launch* and offer to start it. Never end the conversation leaving a completed campaign un-launched without the user having explicitly chosen to leave it paused.

## Step 14 — Statistics connection + follow-up

Two short closing messages to the user:

1. **Engagement statistics (Unipile):** to see likes/replies received on the AI's comments, the LinkedIn account must be connected for statistics. If `list_li_accounts` showed the account is not connected for stats, tell the user — in plain language — that they can connect it on the website **https://app.liseller.com**, in the **Analytics** tab (https://app.liseller.com/pages/analytics) and **Inbox** (https://app.liseller.com/pages/communications). Without it, the campaign still runs and comments, but the weekly audit can't report engagement.
2. **Follow-up:** "I'll plan to audit this in 7 days — run the campaign audit then to validate the setup." (If they ask, point at the `linkedin-campaign-audit` skill.)

---

**Milestone 6 done.** If the user wants to build another campaign, return to `references/milestone-0-context-and-goal.md` and start a fresh pass — keeping every campaign's own focused ~30 keywords and your session ledger up to date.
