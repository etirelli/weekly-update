---
name: weekly-update
description: "Capture team status updates into Confluence weekly report pages under the RHODS Weekly Updates space. Categorizes entries into Highlights (Customer Conversations, Conference Talks, Blog Posts, Team Changes, Major Risks, Open Source News) or per-team Updates. Run whenever there is something worth reporting. Supports free-form input for quick capture or conversational mode for guided entry."
allowed-tools:
  - mcp__atlassian__confluence_create_page
  - mcp__atlassian__confluence_update_page
  - mcp__atlassian__confluence_get_page
  - mcp__atlassian__confluence_get_page_children
  - mcp__atlassian__confluence_search
  - mcp__google-calendar__get-current-time
  - Bash
  - Read
  - Write
argument-hint: "<update description> — e.g., 'Bryan presented Ray+Docling integration at PyTorch Conference Europe'"
---

## Configuration

Configuration is stored in `~/.claude/weekly-update.json`. This file is created automatically on first run.

- **Config file**: `~/.claude/weekly-update.json`
- **Reporting week**: Friday through Thursday
- **Child page title format**: `Weekly Update — YYYY-MM-DD to YYYY-MM-DD` (Friday to Thursday dates)

## EXECUTE NOW

**Target: $ARGUMENTS**

### Step 1: Load configuration

Read the config file at `~/.claude/weekly-update.json`.

```bash
cat ~/.claude/weekly-update.json 2>/dev/null
```

**If the file exists and contains valid JSON with `parent_page_id` and `space_key`:** Use those values. Proceed to Step 2.

**If the file does not exist or is missing required fields:** Run first-time setup:

1. Ask the user: "This is your first time running /weekly-update. Please provide the Confluence page URL or page ID for the parent page where weekly updates will be created as child pages."
2. If the user provides a URL like `https://redhat.atlassian.net/wiki/spaces/SPACE/pages/PAGEID/...`, extract the page ID and space key from the URL.
3. If the user provides a tinylink like `https://redhat.atlassian.net/wiki/x/ENCODED`, decode the page ID:
   ```bash
   encoded="ENCODED"
   echo -n "${encoded}==" | base64 -D 2>/dev/null | xxd -p | python3 -c "import sys; h=sys.stdin.read().strip(); le=int.from_bytes(bytes.fromhex(h)[::-1],'big'); print(le)"
   ```
4. If the user provides just a numeric page ID, use it directly and ask for the space key.
5. **Validate** by calling `mcp__atlassian__confluence_get_page` with the resolved `page_id`. If it fails, report the error and ask the user to try again. If it succeeds, extract the `space_key` from the response metadata.
6. **Save** the configuration:
   ```json
   {
     "parent_page_id": "<resolved_page_id>",
     "space_key": "<space_key>",
     "parent_page_url": "<full_url>"
   }
   ```
   Write this to `~/.claude/weekly-update.json`.
7. Confirm: "Configuration saved. Weekly update pages will be created under: [page title] (space: [space_key]). You can reconfigure anytime by deleting ~/.claude/weekly-update.json and running the skill again."

### Step 2: Get current date

Call `mcp__google-calendar__get-current-time`. This is MANDATORY. Never assume the current date.

Extract the current date from the response.

### Step 2: Compute reporting week boundaries

The reporting week runs **Friday to Thursday**. Calculate the start (Friday) and end (Thursday) of the current reporting week.

Use bash to compute the week boundaries:

```bash
current_date="YYYY-MM-DD"  # from Step 1
current_dow=$(date -j -f "%Y-%m-%d" "$current_date" "+%u")  # 1=Mon..7=Sun
# Friday = day 5
if [ "$current_dow" -ge 5 ]; then
  days_back=$((current_dow - 5))
else
  days_back=$((current_dow + 2))
fi
friday=$(date -j -v-${days_back}d -f "%Y-%m-%d" "$current_date" "+%Y-%m-%d")
thursday=$(date -j -v+6d -f "%Y-%m-%d" "$friday" "+%Y-%m-%d")
echo "week_start=$friday week_end=$thursday"
echo "title=Weekly Update — $friday to $thursday"
```

Store the computed `friday`, `thursday`, and page title for use in later steps.

### Step 3: Parse user input

**If arguments provided:** Use `$ARGUMENTS` as the update description.

**If no arguments:** Ask the user: "What would you like to report? Describe a customer conversation, conference talk, blog post, team change, risk, open source news, or a team-level update."

From the input, determine:
- What happened (the core update)
- Which team/component it relates to
- Who should be attributed (ask if not obvious; default to current user)
- Any relevant links already mentioned
- Whether this is future-dated (look for "next week", "planned for", "will release", "upcoming", specific future dates)

### Step 4: Validate completeness

Analyze the update and proactively ask for any missing information that would make the entry complete for an executive audience. Only ask for information that is genuinely missing — do not over-interrogate well-articulated updates.

**Category-specific checks:**

| Category | Required context | Ask if missing |
|----------|-----------------|----------------|
| Blog Posts | Link to the published blog post | "Can you share the URL of the blog post?" |
| Conference Talks | Conference name + link to conference/abstract | "What conference? Do you have a link to the talk abstract or conference page?" |
| Customer Conversations | Customer name, outcome or next steps | "Which customer? What was the outcome or agreed next step?" |
| Product Releases | Version number, release date, key changes | "What version? What are the key user-facing changes?" |
| Open Source News | Project name, upstream link (PR/issue/release) | "Do you have a link to the upstream PR, issue, or release?" |
| Major Risks | Impact scope, mitigation status | "What's the impact scope? Is there a mitigation plan?" |
| Team Changes | Person name, role, effective date | "Who is changing roles? What's the new role and effective date?" |
| Team Updates | Jira/PR links where relevant, measurable outcome | "Do you have a link to the relevant ticket or PR?" |

**General checks (all entries):**
- Does the entry state a **result or impact**, not just an activity? If it reads like "working on X", ask: "What was the outcome or milestone achieved?"
- Is it specific enough? "Made progress on performance" is too vague — ask: "Can you quantify the improvement or describe what was delivered?"
- Are relevant links included? If the update references a document, PR, Jira ticket, blog, or external resource without a link, ask for it.

### Step 5: Categorize the entry

Determine where the entry belongs using this decision tree:

1. Customer interaction, demo, POC, partnership engagement → **Highlights > Customer Conversations**
2. Conference talk, presentation at an event, speaking engagement → **Highlights > Conference Talks**
3. Blog post, article, or publication → **Highlights > Blog Posts**
4. Hire, departure, team reorganization, role change → **Highlights > Team Changes**
5. Risk, blocker, major incident, significant strategic development → **Highlights > Major Risks / Developments**
6. Upstream contribution, open source release, community engagement → **Highlights > Open Source News**
7. Team-specific work (feature delivery, sprint outcome, technical milestone) → **Team Updates > [Team Name]**
8. Doesn't clearly fit any category → ask the user which section it belongs in

**Dual categorization:** If an entry fits both a Highlight category AND a Team Update (e.g., "AI Pipelines team presented at PyTorch Conference"), ask the user: "This is relevant to both [Highlights > Conference Talks] and [Team Updates > AI Pipelines]. Should I add it to both?"

**Ad-hoc categories:** If an entry doesn't fit any of the six default Highlight categories but is clearly a cross-cutting highlight (not a team update), create a new `### Category Name` subsection under Highlights, placed after the default six categories and before the `---` separator.

### Step 6: Format the entry

Transform the user's input into a polished entry following these rules:

- **Prefix** with `[Component]` tag where relevant (e.g., `[MLflow]`, `[AI Pipelines]`, `[Ray]`)
- **One paragraph**, objective, focused on **results and impact** — NOT execution details
- **Include links** inline (Jira tickets, docs, blog posts, PRs, conference pages)
- **Attribution** at the end: `(Name)`
- Write for an **executive audience** — the wider organization and leadership

**Quality gate — if the input looks like a Jira dump** (e.g., "RHOAIENG-52502 GA readiness tasks completed - multi-arch support"), rewrite it to be executive-appropriate. If you cannot infer the impact, ask the user: "Can you describe the business impact or outcome? The current description reads like a task list."

**Good examples:**
- `[Ray] Bryan Keane demoed our ongoing Ray Data and Docling integration work at PyTorch Conference Europe. (Bryan)`
- `[AI Pipelines] The RHOAI dashboard team completed a multi-month effort to modernize the upstream Kubeflow Pipelines UI, enabling significantly higher upstream feature velocity and unblocking the modular architecture implementation. [Blog post](https://blog.kubeflow.org/modernizing-kubeflow-pipelines-ui/). (Matt)`
- `[MLflow] We are engaged in design work with the MLflow community for a centralized asset registry for MCP servers, agents, and skills. The initial Skills Registry is expected in the May upstream MLflow release. (Edson)`
- `[Feature Store] Nikhil Kathole published a new [Feast Production Deployment Topologies](https://docs.feast.dev/master/how-to-guides/production-deployment-topologies) guide. Field enablement material is being created to support customer deployment conversations. (Nikhil)`

**Bad examples (do NOT produce entries like these):**
- `RHOAIENG-52502 GA readiness tasks completed - multi-arch support, security requirements` — Jira ticket dump, no impact framing
- `Bug fixes - maas-api CrashLoopBackOff from missing db secret (RHOAIENG-53410)` — too granular, execution detail
- `Did some JIRA cleanup - 45 tickets updated` — not meaningful for executives

### Step 7: Find or create the weekly page

**7a. Check for existing page:**

Call `mcp__atlassian__confluence_get_page_children` with:
- `parent_id`: the `parent_page_id` from the config file
- `limit`: `10`
- `include_content`: `false`

Search the returned children for a page whose title matches the computed week title from Step 2.

**7b. If the page exists:**

Call `mcp__atlassian__confluence_get_page` with:
- `page_id`: the found page's ID
- `convert_to_markdown`: `true`

Store the page content and page ID for updating in Step 8.

**7c. If the page does NOT exist:**

Create it using `mcp__atlassian__confluence_create_page` with:
- `space_key`: the `space_key` from the config file
- `parent_id`: the `parent_page_id` from the config file
- `title`: the computed week title (e.g., `"Weekly Update — 2026-04-17 to 2026-04-23"`)
- `content_format`: `"markdown"`
- `content`: the page template below, with the entry already inserted in the correct section

**Page template:**

```markdown
# Weekly Update — {friday} to {thursday}

## Highlights

### Customer Conversations

- *(no entries yet)*

### Conference Talks

- *(no entries yet)*

### Blog Posts

- *(no entries yet)*

### Team Changes

- *(no entries yet)*

### Major Risks / Developments

- *(no entries yet)*

### Open Source News

- *(no entries yet)*

---

## Team Updates

*(Teams appear below as updates are added)*
```

When creating a new page with the first entry, replace the placeholder `*(no entries yet)*` in the appropriate section with the actual entry, or add the team subsection under Team Updates.

**7d. After creating a NEW weekly page, update the landing page:**

This step runs ONLY when a new weekly page was created in step 7c (not when an existing page was found in 7b).

1. Read the landing (parent) page content:
   Call `mcp__atlassian__confluence_get_page` with the `parent_page_id` from config, `convert_to_markdown: true`.

2. Determine the **ISO week number** from the Friday date:
   ```bash
   week_num=$(date -j -f "%Y-%m-%d" "$friday" "+%V")
   year=$(date -j -f "%Y-%m-%d" "$friday" "+%Y")
   ```

3. Construct the new table row:
   ```
   | Week {week_num} | {friday formatted as Mon DD} – {thursday formatted as Mon DD} | [Weekly Update — {friday} to {thursday}]({new_page_url}) |
   ```

4. Parse the landing page markdown:
   - Look for a `## {year}` section header. If it exists, append the new row to the table in that section (at the end, before the next `##` header or end of content).
   - If no `## {year}` section exists, create one with a new table:
     ```markdown
     ## {year}

     | Week | Dates | Link |
     |------|-------|------|
     | Week {week_num} | ... | ... |
     ```
     Add the new year section in chronological order (newest year first, just after the `---` separator that follows Quick Links).

5. Update the landing page:
   Call `mcp__atlassian__confluence_update_page` with:
   - `page_id`: the `parent_page_id`
   - `title`: `"Weekly Updates"` (keep unchanged)
   - `content`: the updated markdown
   - `content_format`: `"markdown"`
   - `is_minor_edit`: `true`
   - `version_comment`: `"Added link for week of {friday} to {thursday}"`

### Step 8: Insert entry into the page

**If updating an existing page:**

1. Parse the page content by section headers (`##` and `###`) to find the target section.
2. Locate the insertion point:
   - For **Highlight categories**: Find the `### Category Name` header. If the section contains only `*(no entries yet)*`, replace the placeholder with the new bullet. Otherwise, append the new bullet after the last existing bullet in that section (before the next `###` header or `---` separator).
   - For **Team Updates**: Find `### Team Name` under `## Team Updates`. If it does not exist, create a new `### Team Name` subsection at the end of the Team Updates section. Append the bullet.
3. Format the entry as a markdown bullet: `- [Component] Description text. (Name)`
4. **Never modify or remove other people's entries.** Only append.

Call `mcp__atlassian__confluence_update_page` with:
- `page_id`: the existing page ID
- `title`: keep the existing title (unchanged)
- `content`: the updated markdown with the new entry inserted
- `content_format`: `"markdown"`
- `is_minor_edit`: `true`
- `version_comment`: `"Added [section > category] entry via /weekly-update"`

### Step 9: Handle future-dated entries

If the entry describes something happening in a **future reporting week** (detected in Step 3):

1. **Determine the target date** of the future event.
2. **Compute which reporting week** the target date falls in (using the same Friday-Thursday logic from Step 2).
3. **If the target date falls within the CURRENT reporting week:** Add one entry with present-tense framing. No dual posting needed.
4. **If the target date falls in a FUTURE reporting week:**
   - **Current week's page**: Add an entry with **forward-looking framing** (e.g., "AI Pipelines is preparing for the v3.4 release, planned for YYYY-MM-DD")
   - **Future week's page**: Add an entry with **completion framing** (e.g., "AI Pipelines released v3.4 on YYYY-MM-DD")
   - Find or create the future week's page using the same logic from Step 7.
5. Inform the user that entries were added to both weeks.

### Step 10: Confirm

Report what was done:

```
Entry added to: Weekly Update — YYYY-MM-DD to YYYY-MM-DD
Section: [Highlights > Category Name | Team Updates > Team Name]
Page: {parent_page_url}/pages/{page_id}

- [Component] The formatted entry text. (Name)
```

If a future-dated entry was also added:

```
Also added forward-looking entry to: Weekly Update — YYYY-MM-DD to YYYY-MM-DD
Page: {parent_page_url}/pages/{page_id}
```

## Reconfiguration

To change the target Confluence page, delete the config file and run the skill again:

```bash
rm ~/.claude/weekly-update.json
```

The skill will prompt for a new parent page on the next invocation.
