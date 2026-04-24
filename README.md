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
         "command": "uvx",
         "args": ["mcp-atlassian"],
         "env": {
           "CONFLUENCE_URL": "https://your-company.atlassian.net/wiki",
           "CONFLUENCE_USERNAME": "your.email@company.com",
           "CONFLUENCE_API_TOKEN": "your_api_token",
           "JIRA_URL": "https://your-company.atlassian.net",
           "JIRA_USERNAME": "your.email@company.com",
           "JIRA_API_TOKEN": "your_api_token"
         }
       }
     }
   }
   ```
   Create an API token at [https://id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens). For Server/Data Center deployments, use a Personal Access Token instead. See the [mcp-atlassian documentation](https://github.com/sooperset/mcp-atlassian) for full setup details.

## Installation

### Option A: Add as a marketplace (recommended)

This allows automatic updates when the skill is improved.

In Claude Code, run:

```
/plugin marketplace add etirelli/weekly-update
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

On first run, the skill prompts you for the **Confluence parent page** where weekly update child pages will be created. You can provide:
- A full page URL (e.g., `https://your-instance.atlassian.net/wiki/spaces/TEAM/pages/123456789/Weekly+Updates`)
- A Confluence tinylink (e.g., `https://your-instance.atlassian.net/wiki/x/zJWdFw`)
- A numeric page ID (you'll also be asked for the space key)

The skill validates the page exists, then saves the configuration to `~/.claude/weekly-update.json`. Subsequent runs use the saved config silently.

| Setting | Value |
|---------|-------|
| Config file | `~/.claude/weekly-update.json` |
| Reporting week | Friday to Thursday |
| Page title format | `Weekly Update — YYYY-MM-DD to YYYY-MM-DD` |

To reconfigure (e.g., point to a different parent page):

```bash
rm ~/.claude/weekly-update.json
```

The skill will prompt for a new parent page on the next invocation.

## Workflow

1. **Throughout the week** (Fri-Thu): team members run `/weekly-update` whenever they have something to report
2. **The skill** captures, validates, categorizes, formats, and inserts entries into the Confluence page
3. **On Thursday**: the manager reviews and edits the page before forwarding highlights to leadership

## Troubleshooting

**"Cannot reach the parent page"**
- Verify your Atlassian MCP server is configured and authenticated
- Check your config: `cat ~/.claude/weekly-update.json`
- Test by running: `mcp__atlassian__confluence_get_page` with the page_id from your config

**Entries appearing in wrong section**
- The skill auto-categorizes based on content. You can override by specifying the category explicitly (e.g., "Add this to Team Updates > MLflow Core")

**Date issues**
- The skill reads the current date from the local system clock. Verify your system date is correct.

## License

Apache License 2.0
