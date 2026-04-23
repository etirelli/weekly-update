# Weekly Update

A Claude Code skill for collaboratively populating weekly status updates into Confluence. Team members run `/weekly-update` whenever they have something worth reporting, and the skill handles categorization, formatting, and page management automatically.

## What It Does

- **Captures updates** into a shared Confluence page, organized by reporting week (Friday-Thursday)
- **Categorizes entries** into Highlights (cross-cutting) or per-team Updates
- **Validates completeness** by asking for missing links, outcomes, and specifics
- **Formats for executives** with results/impact focus, not execution details
- **Handles future-dated entries** by posting to both current and target week with appropriate framing
- **Manages pages automatically** by creating weekly child pages on demand under a parent page

## Highlight Categories

1. Customer Conversations
2. Conference Talks
3. Blog Posts
4. Team Changes
5. Major Risks / Developments
6. Open Source News

Additional ad-hoc categories can be added when needed. Team Updates are listed separately by team name, and only teams with entries appear on the page.

## Prerequisites

Before installing, ensure you have:

1. **Claude Code** installed and configured
2. **Atlassian MCP server** configured with Confluence access. Add to your MCP configuration:
   ```json
   {
     "mcpServers": {
       "atlassian": {
         "command": "npx",
         "args": ["-y", "@anthropic/mcp-atlassian"]
       }
     }
   }
   ```
   Follow the [Atlassian MCP setup guide](https://github.com/anthropics/mcp-atlassian) for authentication.

3. **Google Calendar MCP server** configured (used for reliable date detection):
   ```json
   {
     "mcpServers": {
       "google-calendar": {
         "command": "npx",
         "args": ["-y", "@anthropic/mcp-google-calendar"]
       }
     }
   }
   ```

## Installation

### Option A: Add as a marketplace (recommended)

This allows automatic updates when the skill is improved.

In Claude Code, run:

```
/plugin marketplace add https://github.com/etirelli/weekly-update
```

Then install the plugin:

```
/plugin install weekly-update
```

### Option B: Clone and use as plugin directory

```bash
git clone https://github.com/etirelli/weekly-update.git ~/.claude/plugins/weekly-update
```

Then add it to your Claude Code settings.

## Usage

### Quick capture (with arguments)

```
/weekly-update Bryan presented Ray+Docling integration at PyTorch Conference Europe
```

The skill will:
1. Determine the current reporting week
2. Categorize the entry (Conference Talks in this case)
3. Ask for any missing context (e.g., link to the conference page)
4. Format it for an executive audience
5. Add it to the correct section of the weekly Confluence page

### Conversational mode (no arguments)

```
/weekly-update
```

The skill will ask you what you'd like to report and guide you through the process.

### Future-dated entries

```
/weekly-update AI Pipelines will release v3.4 next Thursday
```

This creates two entries:
- **Current week**: forward-looking ("AI Pipelines is preparing for the v3.4 release, planned for...")
- **Target week**: completion framing ("AI Pipelines released v3.4 on...")

## Entry Format

Each entry follows this format:

```
- [Component] One paragraph describing the result or impact, with inline links. (Author)
```

**Good examples:**
- `[AI Pipelines] The RHOAI dashboard team completed a multi-month effort to modernize the upstream Kubeflow Pipelines UI, enabling significantly higher upstream feature velocity. [Blog post](https://blog.kubeflow.org/modernizing-kubeflow-pipelines-ui/). (Matt)`
- `[Feature Store] Nikhil Kathole published a new Feast Production Deployment Topologies guide. Field enablement material is being created to support customer deployment conversations. (Nikhil)`

**What NOT to write:**
- Jira ticket dumps (`RHOAIENG-52502 GA readiness tasks completed`)
- Execution details (`Bug fixes - maas-api CrashLoopBackOff from missing db secret`)
- Vague activity (`Made progress on performance improvements`)

The skill will ask you to rephrase if the entry is too execution-focused.

## Configuration

The skill is pre-configured with:

| Setting | Value |
|---------|-------|
| Confluence space | `RHODS` |
| Parent page ID | `396203468` |
| Parent page URL | https://redhat.atlassian.net/wiki/spaces/RHODS/pages/396203468 |
| Reporting week | Friday to Thursday |
| Page title format | `Weekly Update — YYYY-MM-DD to YYYY-MM-DD` |

To change the target Confluence page, edit the Configuration section in `.claude/skills/weekly-update/SKILL.md`.

## Workflow

1. **Throughout the week** (Fri-Thu): team members run `/weekly-update` whenever they have something to report
2. **The skill** captures, validates, categorizes, formats, and inserts entries into the Confluence page
3. **On Thursday**: the manager reviews and edits the page before forwarding highlights to leadership

## Troubleshooting

**"Cannot reach the Weekly Updates parent page"**
- Verify your Atlassian MCP server is configured and authenticated
- Test by running: `mcp__atlassian__confluence_get_page` with page_id `396203468`

**Entries appearing in wrong section**
- The skill auto-categorizes based on content. You can override by specifying the category explicitly (e.g., "Add this to Team Updates > MLflow Core")

**Date issues**
- The skill always uses `mcp__google-calendar__get-current-time` for the current date. Ensure your Google Calendar MCP is configured.

## License

Apache License 2.0
