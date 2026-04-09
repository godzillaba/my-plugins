---
description: Search company knowledge across Notion, Google Docs, Slack, Linear, and OffchainLabs GitHub repos. Trigger on "search company", "company search", "search knowledge", "who knows about".
argument-hint: <search query>
---

# Company Knowledge Search

If $ARGUMENTS is empty, ask the user what they want to search for, then proceed.

## Step 1 — Understand the query

Interpret **$ARGUMENTS** in context. If the user is working in a project directory, briefly examine the codebase (README, package.json, CLAUDE.md, recent git history, directory structure — whatever is quick and relevant) to understand what the project is, what domain it operates in, and what terminology it uses. This helps you:

- Expand ambiguous queries (e.g. "the bridge" → the specific bridge contract/system in this repo)
- Add relevant keywords the user might not have typed (project names, component names, acronyms)
- Understand which sources are most likely to have answers

If the query is self-contained and doesn't benefit from project context, skip this.

## Step 2 — Plan search strategies

For each of the 5 sources below, decide a concrete search strategy tailored to the query. Consider:
- What keywords, phrases, or filters will be most effective for this specific source?
- Should the search be broad or narrow? Should it filter by date, author, status, file type, repo, channel, etc.?
- Does the query even make sense for this source? (e.g. a question about a Slack conversation probably doesn't need a GitHub code search)

Print the strategies to the user in this format before spawning agents:

```
Searching for: "<query>"

- **Notion**: <strategy description>
- **Google Drive**: <strategy description>
- **Slack**: <strategy description>
- **Linear**: <strategy description>
- **GitHub (OffchainLabs)**: <strategy description>
```

## Step 3 — Spawn parallel search agents

Spawn **5 agents in parallel** (all in a single message with 5 Agent tool calls), one per source. Use `run_in_background: true` for all agents.

Pass each agent:
1. The user's original query
2. The specific search strategy you planned for that source
3. The tool reference below so it knows what's available

### Agent 1: Notion

```
You are searching Notion for a user.

**Query**: {{QUERY}}
**Strategy**: {{NOTION_STRATEGY}}

Available tools:
- mcp__claude_ai_Notion__notion-search — search across the workspace and connected sources. Params: query, filters (created_date_range, created_by_user_ids), query_type ("internal" or "user"), page_size (max 25), max_highlight_length. Also supports page_url (search within a page), data_source_url (search within a database data source), teamspace_id, content_search_mode ("workspace_search" or "ai_search").
- mcp__claude_ai_Notion__notion-fetch — fetch a full page, database, or data source by URL or ID. Set include_discussions=true to see discussion counts and inline markers. Databases return data sources with collection:// URLs.
- mcp__claude_ai_Notion__notion-get-comments — get comments/discussions from a page. Params: page_id (required), include_all_blocks, include_resolved, discussion_id.
- mcp__claude_ai_Notion__notion-get-teams — list teams/teamspaces in the workspace. Optional query param to filter by name. Returns team IDs useful for scoping searches.
- mcp__claude_ai_Notion__notion-get-users — list workspace users. Params: query (filter by name/email), user_id (specific user or "self"), page_size, start_cursor.
- mcp__claude_ai_Notion__notion-query-data-sources — query Notion databases using SQL or by view. SQL mode: provide data_source_urls and a SQLite query. View mode: provide a view_url. Get data source URLs from notion-fetch on a database.
- mcp__claude_ai_Notion__notion-query-meeting-notes — query meeting notes with filters on title, attendees, created_time, created_by, last_edited_time, last_edited_by. Returns up to 50 rows.

Execute the strategy. You may run multiple searches, fetch pages, query databases, or read meeting notes as needed.

Return a markdown list of findings. Each entry: **Title** (with URL if available), then a brief snippet or summary. If nothing relevant, return "No Notion results found."
```

### Agent 2: Google Drive

```
You are searching Google Drive for a user.

**Query**: {{QUERY}}
**Strategy**: {{GDRIVE_STRATEGY}}

Available tools:
- mcp__claude_ai_Google_Drive__search_files — search Drive files. Query uses Google Drive syntax: fullText contains, title contains, mimeType, modifiedTime, owner, etc. Combine with and/or. Params: query, pageSize, pageToken, excludeContentSnippets.
- mcp__claude_ai_Google_Drive__read_file_content — read a file's natural-language content by ID. Supports Google Docs, Sheets, Slides, PDFs, Office docs, and images.
- mcp__claude_ai_Google_Drive__download_file_content — download a file as base64. For Google native files, set exportMimeType to choose format. Params: fileId (required), exportMimeType.
- mcp__claude_ai_Google_Drive__get_file_metadata — get metadata about a file (title, type, dates, etc.). Params: fileId (required), excludeContentSnippets.
- mcp__claude_ai_Google_Drive__get_file_permissions — list permissions on a file. Params: fileId (required).
- mcp__claude_ai_Google_Drive__list_recent_files — list recent files sorted by recency, lastModified, or lastModifiedByMe. Params: orderBy, pageSize, pageToken, excludeContentSnippets.

Execute the strategy. You may run multiple searches, read files, check metadata, or browse recent files as needed.

Return a markdown list of findings. Each entry: **Title**, then a brief snippet or summary. If nothing relevant, return "No Google Drive results found."
```

### Agent 3: Slack

```
You are searching Slack for a user.

**Query**: {{QUERY}}
**Strategy**: {{SLACK_STRATEGY}}

Available tools:
- mcp__claude_ai_Slack__slack_search_public — search public channels. Supports modifiers: in:channel, from:user, before/after/on dates, "exact phrases", has:link, has:file, is:thread, content_types (messages, files). Params: query, sort (score/timestamp), sort_dir, limit (max 20), cursor, include_context, max_context_length, content_types, include_bots.
- mcp__claude_ai_Slack__slack_search_public_and_private — search ALL channels including public, private, DMs, and group DMs. Same modifiers and params as slack_search_public, plus channel_types filter. Use this when results from public-only search are insufficient.
- mcp__claude_ai_Slack__slack_search_channels — find channels by name/description. Params: query, limit (max 20), cursor, channel_types (public_channel, private_channel), include_archived.
- mcp__claude_ai_Slack__slack_search_users — search users by name, email, or profile attributes (department, role, title). Params: query, limit (max 20), cursor.
- mcp__claude_ai_Slack__slack_read_thread — read a full thread (parent + replies). Params: channel_id, message_ts (both required), limit (max 1000), oldest, latest, cursor.
- mcp__claude_ai_Slack__slack_read_channel — read recent messages from a channel. Also works with user_id for DM history. Params: channel_id (required), limit (max 100), oldest, latest, cursor.
- mcp__claude_ai_Slack__slack_read_canvas — read a Slack canvas document (returns markdown). Params: canvas_id (required).
- mcp__claude_ai_Slack__slack_read_user_profile — get detailed user profile (contact info, status, timezone, org, role). Params: user_id (optional, defaults to current user), include_locale.

Execute the strategy. You may run multiple searches, read threads, read canvases, look up users, etc.

Return a markdown list of findings. Each entry: **Channel** > **Author** (date), then the message snippet. If nothing relevant, return "No Slack results found."
```

### Agent 4: Linear

```
You are searching Linear for a user.

**Query**: {{QUERY}}
**Strategy**: {{LINEAR_STRATEGY}}

Available tools:

Search & research:
- mcp__claude_ai_Linear__research — natural-language query to Linear. Handles complex queries, search, analytics, multi-step actions. Supports multi-turn via conversationId. Prefer this for broad or complex searches.
- mcp__claude_ai_Linear__search_documentation — search Linear's own documentation for features/usage. Params: query, page.

Fetch by ID:
- mcp__claude_ai_Linear__get_issue — get issue details by ID or identifier (e.g. LIN-123). Params: id, includeRelations, includeCustomerNeeds.
- mcp__claude_ai_Linear__get_document — fetch a document by ID or slug.
- mcp__claude_ai_Linear__get_project — get project details. Params: query (name/ID/slug), includeMembers, includeMilestones, includeResources.
- mcp__claude_ai_Linear__get_initiative — get initiative details. Params: query (ID/name), includeProjects, includeSubInitiatives.
- mcp__claude_ai_Linear__get_milestone — get milestone details. Params: project, query (name/ID).
- mcp__claude_ai_Linear__get_team — get team details. Params: query (UUID/key/name).
- mcp__claude_ai_Linear__get_user — get user details. Params: query (ID/name/email/"me").
- mcp__claude_ai_Linear__get_issue_status — get status details. Params: id, name, team.
- mcp__claude_ai_Linear__get_attachment — get attachment content by ID.
- mcp__claude_ai_Linear__get_status_updates — get project/initiative status updates. Params: type ("project"/"initiative"), project/initiative, user, id, limit, cursor, createdAt, updatedAt.
- mcp__claude_ai_Linear__extract_images — extract/view images from markdown content (e.g. issue descriptions).

List & filter:
- mcp__claude_ai_Linear__list_issues — list/filter issues. Params: query, team, assignee ("me"/name/email), state, label, project, cycle, priority (0-4), parentId, limit (max 250), cursor, orderBy, createdAt, updatedAt.
- mcp__claude_ai_Linear__list_projects — list/filter projects. Params: query, team, state, initiative, label, member, limit, cursor, orderBy, includeMembers, includeMilestones.
- mcp__claude_ai_Linear__list_documents — list/filter documents. Params: query, projectId, initiativeId, creatorId, limit, cursor, orderBy, createdAt, updatedAt.
- mcp__claude_ai_Linear__list_comments — list comments on an issue. Params: issueId (required), limit, cursor, orderBy.
- mcp__claude_ai_Linear__list_initiatives — list/filter initiatives. Params: query, owner, status, parentInitiative, limit, cursor, includeProjects, includeSubInitiatives.
- mcp__claude_ai_Linear__list_teams — list teams. Params: query, limit, cursor.
- mcp__claude_ai_Linear__list_users — list users. Params: query (name/email), team, limit, cursor.
- mcp__claude_ai_Linear__list_customers — list customers. Params: query, owner, status, tier, limit, cursor, includeNeeds.
- mcp__claude_ai_Linear__list_cycles — list cycles for a team. Params: teamId (required), type (current/previous/next).
- mcp__claude_ai_Linear__list_milestones — list milestones in a project. Params: project (required).
- mcp__claude_ai_Linear__list_issue_labels — list issue labels. Params: name, team, limit, cursor.
- mcp__claude_ai_Linear__list_issue_statuses — list issue statuses for a team. Params: team (required).
- mcp__claude_ai_Linear__list_project_labels — list project labels. Params: name, limit, cursor.

Execute the strategy. Use research for broad queries, then drill into specifics with get/list tools as needed.

Return a markdown list of findings. Each entry: **Title** (with issue/project ID if applicable), status, and a brief description. If nothing relevant, return "No Linear results found."
```

### Agent 5: GitHub (OffchainLabs)

```
You are searching OffchainLabs GitHub repos for a user.

**Query**: {{QUERY}}
**Strategy**: {{GITHUB_STRATEGY}}

Available tools (via Bash with `gh` CLI):
- gh search code "<query>" --owner OffchainLabs [--repo] [--filename] [--language] --limit N
- gh search issues "<query>" --owner OffchainLabs [--state open/closed] [--label] --limit N
- gh search prs "<query>" --owner OffchainLabs [--state open/closed/merged] --limit N
- gh search repos "<query>" --owner OffchainLabs --limit N
- All support --json for structured output.

For deeper code search: if the strategy identifies specific repos to search, you can shallow-clone them into /tmp and grep locally. This is faster and more flexible than gh search code for targeted searches.
  git clone --depth 1 https://github.com/OffchainLabs/<repo>.git /tmp/<repo>
Then use Grep/Glob on the cloned repo. Clean up with rm -rf /tmp/<repo> when done.

Execute the strategy. Search code, issues, PRs, repos, or any combination as directed.

Return a markdown list of findings grouped by type (only include types with results). Each entry: **Repo/Path or Title** (with URL if available), then a snippet or description. If nothing relevant, return "No GitHub results found."
```

## Step 4 — Compile results

Once all agents complete, compile a unified report organized by source. Omit sections with no results, but note which sources came up empty at the bottom.

## Step 5 — Offer follow-up

After presenting results, offer to read specific items in full or narrow the search.
