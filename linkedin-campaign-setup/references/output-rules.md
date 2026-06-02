# Output Rules — everything the user reads must be user-friendly

This applies to **every** message and summary you render for the user — clarifying questions, the dry-run
review, cost & runway numbers, the closing follow-up, and anything pulled from these `references/`
(default templates, examples, checklists). Translate before showing. Raw field names and tool names exist
only for tool calls; they must never appear in what the user reads.

**Never show the user:**

- **UUIDs / GUIDs** — campaign ids, target ids, account ids, status ids. Translate by name or omit.
- **Raw booleans** `true` / `false` — say "running" / "paused", "enabled" / "disabled", "yes" / "no",
  "connected" / "not connected".
- **Raw API field names / snake_case keys** — `is_active`, `is_setup_completed`, `ai_assistant_method_id`,
  `skip_reason_code`, `post_source_filtering_type`, `post_limit_per_day`, `prices.premium_comment`, etc.
  Translate every one (e.g. "the LinkedIn account is connected", "setup isn't finished yet",
  "the daily comment limit", "the per-Pro-comment price").
- **MCP tool names** — `mcp__claude_ai_Liseller__create_campaign`, `create_campaign`, `update_campaign`,
  `modify_campaign_targets`, or any other `mcp__…` slug or bare tool name. Refer to actions by what they
  do in plain English ("I'll create the campaign and add the keywords", not "I'll call `create_campaign`").
- **Lookup codes** — `premium`, `exclude_company`, `li-search-posts`, `WaitingForProfile`. Say "Pro",
  "companies only", "keyword search", "waiting on profile data".
- **Domain entity / internal class names** — `DCampaign`, `DOpenaiAssistant`, etc.

Default to plain, user-friendly prose at all times.
