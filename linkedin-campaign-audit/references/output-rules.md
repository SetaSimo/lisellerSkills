# Output Rules — everything the user reads must be user-friendly

This applies to **every** line you render for the user — TL;DR, tables, per-campaign findings,
top-engagement entries, action items, follow-ups — and anything pulled from these `references/`
(including `report-template.md`). Translate before showing. Raw field names and tool names exist
only for tool calls; they must never appear in what the user reads.

**Never show the user:**

- **UUIDs / GUIDs** — campaign ids, target ids, account ids, status ids. Translate by name or omit.
- **Raw booleans** `true` / `false` — say "running" / "paused", "enabled" / "disabled", "yes" / "no",
  "connected" / "not connected".
- **Raw API field names / snake_case keys** — `is_enabled`, `skip_reason_code`,
  `ai_assistant_method_id`, `post_limit_per_day`, `content_filter`, etc. Translate every one (e.g.
  "23 of 30 daily comments used", "73% skipped as off-topic", "the daily comment limit").
- **MCP tool names** — `mcp__claude_ai_Liseller__list_campaigns`, `list_campaigns`,
  `update_campaign`, or any other `mcp__…` slug or bare tool name. Describe the action in plain
  English ("I'll raise the limit", not "I'll call `update_campaign`").
- **Lookup codes** — `premium`, `ContentFilter`, `WaitingForProfile`. Say "Pro", "off-topic skips",
  "waiting on profile data".
- **Domain entity / internal class names**.

Default to plain, user-friendly prose at all times.
