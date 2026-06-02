# Output Rules — everything the user reads must be user-friendly

This applies to **every** line you render for the user — the bottleneck diagnosis, the
recommendations themselves (including the **Template** sections in `recommendation-patterns.md`),
cost estimates, the proposed-diff before applying. Translate before showing. Raw field names and
tool names exist only for tool calls; they must never appear in what the user reads.

**Never show the user:**

- **UUIDs / GUIDs** — campaign ids, target ids, account ids, status ids. Translate by name or omit.
- **Raw booleans** `true` / `false` — say "running" / "paused", "enabled" / "disabled", "yes" / "no".
- **Raw API field names / snake_case keys** — `post_limit_per_day`, `skip_reason_code`,
  `ai_assistant_method_id`, `prices.premium_comment`, `content_filter`, etc. Translate every one
  (e.g. "the daily comment limit", "skipped as off-topic", "the per-Pro-comment price").
- **MCP tool names** — `mcp__claude_ai_Liseller__update_campaign`, `update_campaign`,
  `modify_campaign_targets`, or any other `mcp__…` slug or bare tool name (the `Tool call:` lines in
  `recommendation-patterns.md` are agent-only). Describe the action in plain English ("I'll raise the
  limit").
- **Lookup codes** — `premium`, `ContentFilter`. Say "Pro", "off-topic skips".
- **Domain entity / internal class names**.

Default to plain, user-friendly prose at all times.
