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
- mcp__claude_ai_Notion__notion-search — search across the workspace and connected sources. Params: query, filters (created_date_range, created_by_user_ids), query_type ("internal"), page_size (max 25), max_highlight_length.
- mcp__claude_ai_Notion__notion-fetch — fetch a full page by URL or ID for deeper reading.

Execute the strategy. You may run multiple searches or fetch pages as needed.

Return a markdown list of findings. Each entry: **Title** (with URL if available), then a brief snippet or summary. If nothing relevant, return "No Notion results found."
```

### Agent 2: Google Drive

```
You are searching Google Drive for a user.

**Query**: {{QUERY}}
**Strategy**: {{GDRIVE_STRATEGY}}

Available tools:
- mcp__claude_ai_Google_Drive__search_files — search Drive files. Query uses Google Drive syntax: fullText contains, title contains, mimeType, modifiedTime, owner, etc. Combine with and/or.
- mcp__claude_ai_Google_Drive__read_file_content — read a file's content by ID.

Execute the strategy. You may run multiple searches or read files as needed.

Return a markdown list of findings. Each entry: **Title**, then a brief snippet or summary. If nothing relevant, return "No Google Drive results found."
```

### Agent 3: Slack

```
You are searching Slack for a user.

**Query**: {{QUERY}}
**Strategy**: {{SLACK_STRATEGY}}

Available tools:
- mcp__claude_ai_Slack__slack_search_public — search public channels. Supports modifiers: in:channel, from:user, before/after/on dates, "exact phrases", has:link, has:file, is:thread, content_types (messages, files).
- mcp__claude_ai_Slack__slack_search_channels — find channels by name/description.
- mcp__claude_ai_Slack__slack_read_thread — read a full thread.
- mcp__claude_ai_Slack__slack_read_channel — read recent messages from a channel.

Execute the strategy. You may run multiple searches, read threads, etc.

Return a markdown list of findings. Each entry: **Channel** > **Author** (date), then the message snippet. If nothing relevant, return "No Slack results found."
```

### Agent 4: Linear

```
You are searching Linear for a user.

**Query**: {{QUERY}}
**Strategy**: {{LINEAR_STRATEGY}}

Available tools:
- mcp__claude_ai_Linear__research — natural-language query to Linear. Handles complex queries, search, analytics. Supports multi-turn via conversationId.
- mcp__claude_ai_Linear__get_document — fetch a document by ID.
- mcp__claude_ai_Linear__get_initiative — fetch initiative details.

Execute the strategy. You may ask multiple questions or follow up as needed.

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

Execute the strategy. Search code, issues, PRs, repos, or any combination as directed.

Return a markdown list of findings grouped by type (only include types with results). Each entry: **Repo/Path or Title** (with URL if available), then a snippet or description. If nothing relevant, return "No GitHub results found."
```

## Step 4 — Compile results

Once all agents complete, compile a unified report organized by source. Omit sections with no results, but note which sources came up empty at the bottom.

## Step 5 — Offer follow-up

After presenting results, offer to read specific items in full or narrow the search.
