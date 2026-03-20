---
name: chat-dashboard
description: Fetches and summarizes all of the user's Claude chat history, then renders an interactive searchable dashboard artifact with tag filters, status indicators, and Continue links. Trigger when the user asks to see their chat history, browse past conversations, find a previous chat, or get a dashboard of their Claude chats.
---

# Chat Dashboard Skill

Renders a fully interactive dashboard of the user's complete Claude chat history — searchable, filterable by tag and status, with a "Continue →" link on every card.

## When to trigger

Trigger this skill when the user says things like:
- "show my chat history"
- "give me a dashboard of my chats"
- "what have I been working on"
- "find my previous conversations"
- "summarize all my chats"
- "chat dashboard"

## Steps

### Step 1 — Fetch all chats

Paginate through the user's complete chat history using `recent_chats`:

1. Call `recent_chats` with `n=20`, `sort_order="desc"`
2. Record the `updated_at` of the last result
3. Call again with `before=<that updated_at>`, `n=20`
4. Repeat until a call returns fewer than 20 results
5. Collect every chat object across all pages into a single list

Each chat object has: `uri`, `url`, `updated_at`, and a `Summary` block containing `Title` and a content summary.

### Step 2 — Summarize and tag each chat

For each chat, extract:

- **title**: concise, under 8 words
- **summary**: 2-3 sentences covering what was discussed, decisions made, tools or methods used, and any open questions or next steps. Be specific and technical — not generic.
- **tech**: comma-separated list of tools, frameworks, methods, or domain terms mentioned (max 12 items). Leave blank if none.
- **tags**: 1-3 short lowercase tags that describe the topic of this specific chat. Generate these freely from the content — do not use a fixed list. Good tags are:
  - Specific enough to be useful as a filter (e.g. `rna-seq`, `resume`, `recipe`, `tax`, `travel`)
  - General enough that multiple chats could share the same tag (e.g. `debugging` not `python-list-index-error`)
  - Consistent across chats — if two chats are both about cooking, both should get `cooking`, not one `cooking` and one `food`
  - Lowercase, no spaces (use hyphens for multi-word: `job-search`, `machine-learning`)
  - Based entirely on the user's actual chat content, not on any assumed domain

  After summarizing all chats, review the full tag set and normalize any near-duplicates (e.g. collapse `job-search` and `jobs` into one). The final tag list should reflect the user's real topics, whatever those are.
- **status**:
  - `active` — conversation ended mid-task or has clear unresolved next steps
  - `pending` — waiting on external input, file upload, or user action
  - `resolved` — task completed, question answered, decision made

### Step 3 — Render the dashboard artifact

Once all chats are summarized, render the following React artifact. Replace `INJECT_CHATS_HERE` with the actual JSON array of summarized chat objects.

The artifact must include:
- A search bar that filters across title + summary + tech in real time
- Tag filter pills (only show tags that exist in the data)
- Status filter pills: All / Active / Pending / Resolved
- A responsive card grid — each card shows: title, date, tags, summary, tech (monospace), status dot, Continue → link
- localStorage persistence so re-opening is instant
- A Refresh button that calls `sendPrompt("refresh my chat dashboard")` to re-run the skill

```jsx
import { useState, useEffect } from "react";

// Dynamic tag colors — generated from the actual tags in the user's data
const TAG_PALETTES = [
  { bg: "#E1F5EE", color: "#0F6E56" },
  { bg: "#E6F1FB", color: "#185FA5" },
  { bg: "#EEEDFE", color: "#534AB7" },
  { bg: "#FAECE7", color: "#993C1D" },
  { bg: "#EAF3DE", color: "#3B6D11" },
  { bg: "#FAEEDA", color: "#854F0B" },
  { bg: "#FBEAF0", color: "#993556" },
  { bg: "#F1EFE8", color: "#5F5E5A" },
  { bg: "#E6F1FB", color: "#0C447C" },
  { bg: "#FAECE7", color: "#712B13" },
];
const allUniqueTags = [...new Set(CHATS.flatMap(c => c.tags))].sort();
const TAG_META = Object.fromEntries(
  allUniqueTags.map((tag, i) => [
    tag,
    { label: tag.replace(/-/g, " "), ...TAG_PALETTES[i % TAG_PALETTES.length] }
  ])
);

const STATUS_META = {
  active:   { label: "Active",   color: "#1D9E75" },
  pending:  { label: "Pending",  color: "#BA7517" },
  resolved: { label: "Resolved", color: "#888780" },
};

const STORAGE_KEY = "chat_dashboard_v2";

// ── INJECT YOUR DATA HERE ──────────────────────────────────────────────────
const CHATS = INJECT_CHATS_HERE;
// ──────────────────────────────────────────────────────────────────────────

function Tag({ name }) {
  const m = TAG_META[name] || TAG_META.other;
  return <span style={{ fontSize: 11, padding: "2px 8px", borderRadius: 20, background: m.bg, color: m.color, fontWeight: 500, whiteSpace: "nowrap" }}>{m.label}</span>;
}

function StatusDot({ status }) {
  const m = STATUS_META[status] || STATUS_META.resolved;
  return (
    <span style={{ display: "flex", alignItems: "center", gap: 5, fontSize: 11, color: "var(--color-text-tertiary)" }}>
      <span style={{ width: 6, height: 6, borderRadius: "50%", background: m.color, display: "inline-block" }} />
      {m.label}
    </span>
  );
}

function ChatCard({ chat }) {
  return (
    <div
      style={{ background: "var(--color-background-primary)", border: "0.5px solid var(--color-border-tertiary)", borderRadius: "var(--border-radius-lg)", padding: "1rem 1.25rem", display: "flex", flexDirection: "column", gap: 8, transition: "border-color 0.15s" }}
      onMouseEnter={e => e.currentTarget.style.borderColor = "var(--color-border-secondary)"}
      onMouseLeave={e => e.currentTarget.style.borderColor = "var(--color-border-tertiary)"}
    >
      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "flex-start", gap: 8 }}>
        <span style={{ fontSize: 14, fontWeight: 500, color: "var(--color-text-primary)", lineHeight: 1.4 }}>{chat.title}</span>
        <span style={{ fontSize: 11, color: "var(--color-text-tertiary)", whiteSpace: "nowrap", flexShrink: 0 }}>{chat.date}</span>
      </div>
      <div style={{ display: "flex", gap: 4, flexWrap: "wrap" }}>
        {(chat.tags || []).map(t => <Tag key={t} name={t} />)}
      </div>
      <p style={{ fontSize: 13, color: "var(--color-text-secondary)", lineHeight: 1.6, margin: 0 }}>{chat.summary}</p>
      {chat.tech && (
        <div style={{ fontSize: 11, color: "var(--color-text-tertiary)", fontFamily: "var(--font-mono)", background: "var(--color-background-secondary)", padding: "4px 8px", borderRadius: "var(--border-radius-md)", lineHeight: 1.5 }}>{chat.tech}</div>
      )}
      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginTop: 2 }}>
        <StatusDot status={chat.status} />
        <div style={{ display: "flex", gap: 6 }}>
          <button
            onClick={() => sendPrompt("refresh my chat dashboard")}
            style={{ fontSize: 12, padding: "4px 10px", borderRadius: "var(--border-radius-md)", border: "0.5px solid var(--color-border-secondary)", background: "transparent", color: "var(--color-text-tertiary)", cursor: "pointer" }}
          >Refresh ↻</button>
          {chat.url && (
            <a href={chat.url} target="_blank" rel="noreferrer"
              style={{ fontSize: 12, padding: "4px 10px", borderRadius: "var(--border-radius-md)", border: "0.5px solid var(--color-border-secondary)", background: "transparent", color: "var(--color-text-secondary)", textDecoration: "none" }}
              onMouseEnter={e => { e.currentTarget.style.background = "var(--color-background-secondary)"; e.currentTarget.style.color = "var(--color-text-primary)"; }}
              onMouseLeave={e => { e.currentTarget.style.background = "transparent"; e.currentTarget.style.color = "var(--color-text-secondary)"; }}
            >Continue →</a>
          )}
        </div>
      </div>
    </div>
  );
}

function Pill({ label, active, onClick }) {
  return (
    <button onClick={onClick} style={{
      fontSize: 12, padding: "4px 10px", borderRadius: 20, cursor: "pointer",
      border: `0.5px solid ${active ? "var(--color-border-info)" : "var(--color-border-secondary)"}`,
      background: active ? "var(--color-background-info)" : "var(--color-background-primary)",
      color: active ? "var(--color-text-info)" : "var(--color-text-secondary)",
    }}>{label}</button>
  );
}

export default function App() {
  const [search, setSearch] = useState("");
  const [activeTag, setActiveTag] = useState("all");
  const [activeStatus, setActiveStatus] = useState("all");

  const allTags = [...new Set(CHATS.flatMap(c => c.tags || []))].sort();

  const filtered = CHATS.filter(c => {
    if (activeTag !== "all" && !(c.tags || []).includes(activeTag)) return false;
    if (activeStatus !== "all" && c.status !== activeStatus) return false;
    if (search) {
      const q = search.toLowerCase();
      if (![c.title, c.summary, c.tech].join(" ").toLowerCase().includes(q)) return false;
    }
    return true;
  });

  return (
    <div style={{ padding: "1rem 0", fontFamily: "var(--font-sans)" }}>
      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "flex-start", marginBottom: "1.25rem", flexWrap: "wrap", gap: 8 }}>
        <div>
          <div style={{ fontSize: 18, fontWeight: 500, color: "var(--color-text-primary)" }}>Chat history</div>
          <div style={{ fontSize: 13, color: "var(--color-text-tertiary)", marginTop: 2 }}>{CHATS.length} chats indexed</div>
        </div>
      </div>

      <input type="text" value={search} onChange={e => setSearch(e.target.value)}
        placeholder="Search titles, summaries, tools..."
        style={{ width: "100%", fontSize: 13, padding: "8px 12px", marginBottom: 10, borderRadius: "var(--border-radius-md)", border: "0.5px solid var(--color-border-secondary)", background: "var(--color-background-secondary)", color: "var(--color-text-primary)", outline: "none" }} />

      <div style={{ display: "flex", gap: 6, flexWrap: "wrap", alignItems: "center", marginBottom: 6 }}>
        <span style={{ fontSize: 12, color: "var(--color-text-tertiary)" }}>Tag:</span>
        <Pill label="All" active={activeTag === "all"} onClick={() => setActiveTag("all")} />
        {allTags.map(t => <Pill key={t} label={TAG_META[t]?.label || t} active={activeTag === t} onClick={() => setActiveTag(t)} />)}
      </div>

      <div style={{ display: "flex", gap: 6, flexWrap: "wrap", alignItems: "center", marginBottom: "1rem" }}>
        <span style={{ fontSize: 12, color: "var(--color-text-tertiary)" }}>Status:</span>
        {["all", "active", "pending", "resolved"].map(s => (
          <Pill key={s} label={s === "all" ? "All" : STATUS_META[s].label} active={activeStatus === s} onClick={() => setActiveStatus(s)} />
        ))}
        <span style={{ fontSize: 12, color: "var(--color-text-tertiary)", marginLeft: "auto" }}>{filtered.length} of {CHATS.length}</span>
      </div>

      {filtered.length > 0
        ? <div style={{ display: "grid", gridTemplateColumns: "repeat(auto-fill, minmax(300px, 1fr))", gap: 12 }}>
            {filtered.map(c => <ChatCard key={c.id} chat={c} />)}
          </div>
        : <div style={{ textAlign: "center", padding: "2rem", color: "var(--color-text-tertiary)", fontSize: 13, border: "0.5px dashed var(--color-border-secondary)", borderRadius: "var(--border-radius-lg)" }}>
            No chats match your filters.
          </div>
      }
    </div>
  );
}
```

## Output format

Before rendering the artifact, briefly confirm how many chats were found:

> "Found X chats across Y pages. Rendering your dashboard..."

Then render the artifact with the real data injected. Do not summarize or list the chats in text — the artifact is the output.

## Refresh handling

When the user says "refresh my chat dashboard" (triggered by the Refresh button in the artifact), re-run this skill from Step 1 to fetch and re-render with the latest data.

## Notes

- This skill requires `recent_chats` which is only available inside claude.ai. It will not work in Claude Code or the API.
- For large chat histories (100+), inform the user it may take a moment and show progress as pages are fetched.
- If a chat has no title or summary, use the date as the title and tag it as `other` with status `resolved`.
