# Liseller Skills

**Your AI shouldn't leave "Great post! 🚀" on your buyers' LinkedIn.**

Liseller puts you in the comment sections where your ICP already hangs out — finding their posts, drafting replies in your voice, every day, inside safe limits. This repo is the set of Claude skills that drive that workflow. Point any MCP-compatible client (Claude, Claude Code, Cursor, VS Code, ChatGPT) at the Liseller MCP server, install these skills, and your agent runs LinkedIn engagement the way Liseller's best operators do.

Three things they handle:

- **ICP targeting** — keyword and profile targets built around who you actually sell to, not "everyone in tech." Find the right profiles, kill the dead keywords.
- **AI comment drafting** — persona-driven replies that read like you wrote them, not like a bot. The right voice on the right posts.
- **Daily engagement** — campaigns run on a schedule within rate limits, then the skills audit what happened and tell you what to change.

No "thought leadership" theater. You show up in the right comment sections, the agent reads the numbers, you scale what works.

---

## Quickstart

Two parts, every client: **connect the MCP server**, then **install the skills**. The MCP server is the connection to your Liseller account; the skills are the playbooks the agent follows once it's connected.

### 1. Connect the Liseller MCP server

Server URL: `https://mcp.liseller.com/mcp`

No token or OAuth app setup. On the first MCP call your browser opens to authorize access to Liseller — approve it once and you're done.

**Claude (desktop)**
Open Claude connectors (the server URL is copied automatically), paste it into the **Add custom connector** dialog, then approve access on the Liseller screen. Leave Advanced settings empty.

**Claude Code**
```bash
claude mcp add --transport http liseller https://mcp.liseller.com/mcp
claude /mcp
```

**Cursor** — click **Install in Cursor**, or add to `~/.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "liseller": {
      "url": "https://mcp.liseller.com/mcp"
    }
  }
}
```

**VS Code** — click **Install in VS Code**, or add to `.vscode/mcp.json`:
```json
{
  "servers": {
    "liseller": {
      "type": "http",
      "url": "https://mcp.liseller.com/mcp"
    }
  }
}
```

**ChatGPT**
Open ChatGPT connectors, click **Add custom connector**, paste `https://mcp.liseller.com/mcp`, and authorize access to Liseller. Requires a paid plan (Plus, Pro, Business, Enterprise, or Edu).

**Other MCP clients** — add to your client config (OAuth on first call where supported):
```json
{
  "mcpServers": {
    "liseller": {
      "url": "https://mcp.liseller.com/mcp"
    }
  }
}
```

### 2. Install the skills

Open a terminal in your workspace or project and run:
```bash
npx skills add https://github.com/SetaSimo/lisellerSkills --yes
```
Then reload your client so it picks up the new skills.

> Manually added skills don't auto-update — re-run the command to refresh them.

### 3. Try it

Once both are connected, just ask:

- "Set up a new LinkedIn campaign targeting fractional CMOs."
- "Run a weekly audit on my campaigns."
- "My campaign's been flat for two weeks — how do I grow it?"

The right skill triggers automatically.

---

## Skills

### `linkedin-campaign-setup`
Builds a new campaign from scratch, correctly the first time. Walks through campaign type (by keyword or by profile), ICP-driven target design (including finding profiles that match your ICP), content and profile filters, AI method, persona and voice, schedule, limits with ramp-up, and a dry-run review before anything goes live. Built to set up several campaigns back-to-back in one session, and to stop the classic mistake of switching a campaign on before its targets are validated.

**Use it when** you want to create a campaign, you're describing a new audience to reach, or you're migrating from another tool.

### `linkedin-campaign-audit`
A read-only weekly performance review. Pulls throughput vs. limits, the skip-reason breakdown (why posts got passed over — off-topic, filtered, out of limit, etc.), budget burn and runway, target relevance, AI-method fit, and your top-engaging comments. Produces a structured report: a three-bullet TL;DR, a per-campaign health table, account-level findings, and prioritized action items. It proposes changes but never applies them — you approve, or hand off to the growth skill.

**Use it when** you want a weekly review, something feels off ("why so few comments today?"), or you're establishing a baseline before making changes.

### `linkedin-growth-recommendations`
Scales a campaign that already works. Diagnoses the one bottleneck holding back reach — limit-bound, filter-bound, target-bound, quality-bound, or concentration-bound — then gives up to five specific, numbers-backed moves: raise the daily limit, expand or prune keywords, upgrade the AI method, diversify targets. Each recommendation names the exact parameter, the value, and the expected daily cost. Ends with a measurement plan so you change one or two things at a time and actually know what moved the needle.

**Use it when** a campaign with 2+ weeks of history has plateaued, you want more reach, or you're acting on findings from an audit. (For a brand-new or broken campaign, run setup or audit first.)

---

## Requirements

- An MCP-compatible client (Claude, Claude Code, Cursor, VS Code, ChatGPT on a paid plan, or any client that speaks MCP).
- A Liseller account — authorized on first use via your browser.
- `npx` available to install the skills.
