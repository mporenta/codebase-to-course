# Module 5: When Context Runs Dry

### Teaching Arc
- **Metaphor:** A field reporter's notebook. The reporter has a fixed-size notebook — say 200 pages. After two weeks of interviews, the notebook is nearly full. They can't just throw old pages away (they need the context), so they tear out the first 150 pages and replace them with a single page of bullet-pointed highlights: "Met source A, said X. Visited site B, observed Y." The last 50 pages of detailed notes stay. New entries continue from the bullet summary forward. This is exactly what pi does to the model's memory.
- **Opening hook:** Ever notice how, in a really long pi session, the model sometimes "forgets" what you said an hour ago — but remembers the file it edited five minutes ago? That's compaction. Pi is consciously sacrificing old details to keep the conversation alive.
- **Key insight:** Language models have a **context window** — a hard limit on how much text they can consider at once (measured in tokens). When a conversation grows close to that limit, pi runs **compaction**: it sends the older messages to the model, asks for a summary, and replaces those old messages with the summary plus a record of what files were touched.
- **"Why should I care?":**
  - **It explains weird "amnesia" mid-session** — if pi suddenly seems to have forgotten an early decision, compaction happened and the summary may have lost a detail.
  - **You can steer it** — `/compact` lets you trigger it manually and add instructions like "make sure the summary preserves the API contract we agreed on."
  - **You can budget for it** — knowing the context window matters when you decide how big a file to paste or how long a session to keep open.

### Course-Wide Context

Module 5 of 6. Module 4 explained sessions as trees. This module explains why a tree can't grow infinitely from the model's perspective — and how pi handles that limit.

**Course title:** Inside Pi: How a Coding Agent Actually Works
**Accent color:** vermillion
**Background:** Module 5 is odd → `--color-bg-warm`

### Screens (5 recommended)

1. **The context window** — Open with the constraint. Every model has a maximum number of tokens it can see at once. Sonnet has ~200K tokens. Smaller models have less. The system prompt + every user message + every assistant message + every tool result counts toward that limit. When you blow past it, the API rejects the request.

2. **Two triggers, two responses** — Pi runs compaction in two situations:
   - **Threshold** (proactive) — after each turn, pi checks if `context_used > window - reserve`. If yes, compact before the next turn so we don't crash.
   - **Overflow recovery** (reactive) — if the model returns an "out of context" error anyway, pi removes the error message, compacts, and retries the request automatically.
   Show the `shouldCompact` snippet here. Code↔English translation.

3. **What compaction actually does** — Walk through the algorithm:
   - **Find a cut point** — walk backward from the newest message until ~20K tokens of recent context are kept.
   - **Bundle the older messages** — everything before the cut.
   - **Ask the model to summarize** — pi makes a separate LLM call: "summarize this conversation, preserving key decisions and file changes."
   - **Track files** — pi separately scans the compacted messages for tool calls, building a list of files that were read or modified.
   - **Write a CompactionEntry** — store the summary + file list as a single entry in the JSONL tree.
   - **Replace the past** — next turn, the model sees: system prompt + summary entry + last 20K tokens of detailed messages.
   Use numbered step cards or a flow animation.

4. **Visual: before and after** — Use the layer toggle or a side-by-side comparison: "Before compaction" shows 60K tokens spread across many messages; "After compaction" shows a 2K-token summary + 20K tokens of recent messages. Total is now well under the limit.

5. **Quiz**

### Code Snippets (pre-extracted)

**Snippet A — The compaction trigger function (from `packages/coding-agent/src/core/compaction/compaction.ts` lines 219-222):**
```
export function shouldCompact(contextTokens: number, contextWindow: number, settings: CompactionSettings): boolean {
    if (!settings.enabled) return false;
    return contextTokens > contextWindow - settings.reserveTokens;
}
```

**Snippet B — The compaction file-tracking interface (from `packages/coding-agent/src/core/compaction/compaction.ts` lines 33-36):**
```
export interface CompactionDetails {
    readFiles: string[];
    modifiedFiles: string[];
}
```

**Snippet C — A CompactionEntry on disk, conceptual (matching `packages/coding-agent/docs/session-format.md`):**
```
{"type":"compaction","id":"cpt-1","parentId":"msg-42","timestamp":"2025-05-16T15:20:00Z","summary":"User wanted to migrate from SQLite to Postgres. We discussed schema differences, agreed to keep table names but change column types. Read schema.sql, migrations.sql, db.ts. Modified db.ts and added migrations/2025-05-16-init.sql. Final decision: use jsonb for the metadata column.","firstKeptEntryId":"msg-32","tokensBefore":58000,"details":{"readFiles":["schema.sql","migrations.sql","db.ts"],"modifiedFiles":["db.ts","migrations/2025-05-16-init.sql"]}}
```

### Interactive Elements

- [x] **Code↔English translation (Snippet A — shouldCompact)** — Walk through:
  - `export function shouldCompact(...)` → "A simple function that answers yes or no: do we need to compact right now?"
  - `if (!settings.enabled) return false` → "If the user turned auto-compaction off in settings, never trigger it."
  - `return contextTokens > contextWindow - settings.reserveTokens` → "Otherwise, return true whenever the tokens we are using have crossed past the limit minus a safety reserve. The reserve (default 16,000 tokens) leaves room for the model's reply."

- [x] **Code↔English translation (Snippet B)** — The file-tracking shape. "When pi compacts, it doesn't just produce prose — it also remembers exactly which files were touched in the compacted section. `readFiles` are files the agent looked at; `modifiedFiles` are files it changed. This lets the model — and you — know what's been worked on even after the detailed messages are gone."

- [x] **Code↔English translation (Snippet C)** — A real compaction entry. Walk through `type`, `summary`, `firstKeptEntryId` (where the un-summarized recent messages start), `tokensBefore` (how big the context was when compaction ran), `details.readFiles` / `details.modifiedFiles`.

- [x] **Data flow animation (REQUIRED — at least one in the whole course)** — Title: "Anatomy of a compaction." Actors:
  1. **Old messages** (notebook-pages icon)
  2. **Pi** (gear icon)
  3. **The Model** (brain icon, used for summarization)
  4. **CompactionEntry** (scroll icon)
  Steps (NO apostrophes in labels):
  - "Pi checks token usage after each turn" (highlight Pi)
  - "Threshold crossed — pi marks the cut point" (highlight Old messages, packet Pi → Old messages)
  - "Pi sends the old chunk to the model with a summarize instruction" (highlight Model, packet Old messages → Model)
  - "Model writes a structured summary" (highlight Model)
  - "Pi extracts which files were read or modified" (highlight Pi, packet Model → Pi)
  - "Pi writes a single CompactionEntry to the session file" (highlight CompactionEntry, packet Pi → CompactionEntry)
  - "Next turn, the model sees the summary plus the recent messages — no more overflow" (highlight CompactionEntry)

- [x] **Numbered step cards** — The 5-step compaction process (cut point → bundle → summarize → track files → replace).

- [x] **Pattern cards** — Two cards side by side:
  - **Threshold compaction** — Runs proactively before each LLM call when context is nearly full. Default trigger: context > window − 16K reserve.
  - **Overflow recovery** — Runs reactively when the model has already crashed with "context too large." Pi removes the error message and retries with a fresh compaction.

- [x] **Quiz** — 3-4 scenario questions:
  1. "An hour into a debugging session, pi suddenly can't remember the file path you mentioned at the start. What likely happened?" (Correct: compaction. The path was in the early messages, those got summarized, and the summary didn't include that exact detail. Steer pi by re-stating the path or running `/compact "preserve the schema.sql path"`.)
  2. "You're about to run a complicated migration and you'd like the summary so far to be carefully worded — capturing every API contract you agreed on. How would you do that?" (Correct: run `/compact "preserve all API contracts and the agreed-on migration order"`. This passes custom instructions to the summarization call.)
  3. "You paste a 100,000-token file into pi and ask it to refactor. What's likely to happen?" (Correct: pi will fill most of the context window with the file alone, leaving little room for the model to think or call tools. Better to ask pi to read the file in chunks, or split it before pasting.)
  4. "You've added a Postgres extension to pi that exposes some new tools, and you want compaction to preserve information about which database tables were touched. What's the cleanest way to do that?" (Correct: write an extension that hooks `session_before_compact` to add database-table tracking alongside file tracking. The point: compaction is hookable — `details` exists for exactly this.)

- [x] **Callout (aha!)** — "**Compaction is lossy on purpose.** The whole point is to throw information away so the conversation can continue. The model will sometimes lose a detail. That's the trade — and it's why you can manually trigger compaction with custom instructions to preserve what matters to you."

- [x] **Callout (warning)** — "**The original messages aren't deleted from the file** — they're just hidden from the model. You can always scroll back through `/tree` to see what was said before compaction. Compaction is about what the model sees, not what's on disk."

### Glossary Tooltips

- **context window** — The maximum amount of text a language model can consider at once, measured in tokens. Anthropic's Sonnet has about 200,000 tokens; smaller models have less. When a conversation goes over, the model can't respond.
- **token** (re-use from module 1 if already in glossary, else add) — A piece of text the model counts as one unit. Roughly a word or word-piece. "Hello world" is about 2 tokens. Long file paths can be 5-10 tokens each.
- **summarize** (compaction context) — Read a long piece of text and produce a shorter version that captures the important parts. The model is good at this; pi delegates it.
- **reserve** (token reserve) — A safety buffer of tokens pi keeps unused, so the model has room to reply even when the conversation is near the limit. Default 16,000.
- **lossy** — A process that loses some information. JPEG is lossy compression; compaction is lossy summarization. Opposite of "lossless."
- **hook** (extension hook) — A point in pi's lifecycle where extensions can run code. `session_before_compact` is a hook that fires before pi runs compaction; an extension can listen for it and customize the behavior.
- **proactive vs reactive** — Proactive means "do it before the problem happens." Reactive means "wait until the problem happens, then fix it." Pi does both — it tries to compact proactively, but falls back to reactive recovery if a crash happens anyway.
- **migration** (database context) — A scripted change to a database's structure — adding a table, changing a column, etc. The quiz uses this; tooltip it on first use.
- **schema** (database context, if not already in glossary) — The blueprint of a database: what tables exist, what columns each one has.

### Reference Files to Read

- `references/content-philosophy.md` — required, full
- `references/gotchas.md` — required, full
- `references/interactive-elements.md` — sections: "Code ↔ English Translation Blocks", "Multiple-Choice Quizzes", "Message Flow / Data Flow Animation", "Numbered Step Cards", "Pattern/Feature Cards", "Callout Boxes", "Glossary Tooltips"
- `references/design-system.md` — Color Palette

### Connections

- **Previous module:** Module 4 — "Memory as a Tree" — sessions are JSONL trees of every message. This module explains what happens when the tree gets too big for the model to read.
- **Next module:** Module 6 — "Making Pi Yours" — extensions, skills, and the event bus. Tease: "We've seen the machinery. Now let's see how you can bolt onto it — pi is designed from the ground up to be modified, and the surface for doing that is surprisingly small."
- **Tone/style notes:** Vermillion accent; odd module → `--color-bg-warm`. Notebook metaphor only used in this module. No "restaurant" / "switchboard" reuse.
