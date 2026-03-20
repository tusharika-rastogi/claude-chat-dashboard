# claude-chat-dashboard

A Claude skill that fetches your complete chat history, summarizes every conversation with technical detail, and renders a searchable, filterable dashboard inside claude.ai.

## What it does

Run it once and get a full interactive view of everything you've worked on with Claude:

- Paginates through your **entire** chat history, not just the last 20
- Generates a **technical summary** per chat: decisions made, tools used, open questions
- **Auto-tags** each chat based on what is actually in the conversation, no fixed vocabulary
- Shows **status** per chat: Active (unresolved), Pending (waiting on input), or Resolved
- **Search** across titles, summaries, and tech stack in real time
- **Filter** by tag and status
- **Continue** link on every card jumps straight back into that chat

## How to use it

### 1. Install the skill

**Option A: Upload directly in claude.ai**

1. Download this repo as a zip, or clone it
2. In claude.ai, go to **Settings, then Skills**
3. Upload the `chat-dashboard` folder

**Option B: Link a GitHub repo**

1. Fork this repo
2. In claude.ai Settings, then Skills, paste your fork URL

### 2. Run it

Start a new chat and type any of:

```
show my chat dashboard
what have I been working on
give me a summary of all my chats
chat dashboard
```

Claude will fetch all your chats, summarize them, and render the dashboard as an interactive artifact in the conversation.

### 3. Refresh

Type "refresh my chat dashboard" or click the Refresh button inside the dashboard to re-fetch with the latest chats.

## How it works

Claude has a `recent_chats` tool available during live claude.ai conversation turns. This skill instructs Claude to:

1. Paginate `recent_chats` with a `before` cursor until all chats are collected
2. Summarize and tag each one using its built-in knowledge
3. Inject the structured data into a React artifact and render it

This is why the skill only works inside **claude.ai** - the `recent_chats` tool is injected by claude.ai into conversation turns and is not available in the public API or Claude Code.

## Why a skill and not an app

The `recent_chats` tool is session-scoped to the logged-in user and only available during a live claude.ai conversation turn. There is no public API endpoint for it. A standalone app cannot call it on another user's behalf.

A skill is the correct format precisely because it runs inside the conversation turn where the tool is available, and it automatically uses the credentials of whoever is running it, so every user gets their own chat history with no authentication setup.

## Tags

Tags are generated dynamically from your actual chat content, there is no fixed vocabulary. Claude reads each conversation and invents 1-3 short lowercase tags that reflect what was discussed (e.g. `rna-seq`, `resume`, `recipe`, `tax-planning`, `debugging`). After processing all chats, near-duplicate tags are normalized into a single consistent label.

The filter pills in the dashboard are built from whatever tags emerge, so two people with completely different chat histories will see completely different tag sets.

## Requirements

- Claude.ai account (Free, Pro, or Team)
- Skills feature enabled in Settings

## Contributing

PRs welcome. If you improve the summarization prompt, extend the artifact UI, or add new features, open a PR with a brief description of what changed and why.

## License

MIT

---

Built by [Tusharika Rastogi](https://github.com/tusharika-rastogi)
