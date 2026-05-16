# Module 4: Memory as a Tree

### Teaching Arc
- **Metaphor:** A choose-your-own-adventure book — but all the branches are bound in one volume. You read page 1, follow the choice to page 4, then page 7, then page 12. The book also contains an alternate path through pages 5 → 8 → 13 — you didn't take it, but it's still there in the same book. You can flip back to page 4 and choose the other door any time. Pi's session log is exactly like this: one file, multiple paths, the current path is whichever one you're standing on.
- **Opening hook:** When you re-open a long conversation with pi, every message is exactly where you left it. Even better, you can rewind to any earlier point and continue a different way — without losing the original. How does pi pull that off?
- **Key insight:** Pi sessions are not chat logs. They're **trees** stored as JSONL files. Every message has an `id` and a `parentId`. Walking the parent chain from the current "leaf" back to the root reconstructs the path. Branching means a new entry with the same `parentId` as another entry — a fork in the road, preserved forever.
- **"Why should I care?":**
  - **Recovering from a bad turn.** If the model went down a wrong path and made bad edits, you can `/tree` back to before the mistake and try again — without losing the work that came after, because branches don't overwrite each other.
  - **Trying alternatives.** "What if I'd asked it to use Postgres instead of SQLite?" Branch from that question, explore both worlds, compare.
  - **Debugging.** Every model message, every tool call, every result is logged. When something weird happens, the JSONL file is your forensic record.

### Course-Wide Context

Module 4 of 6. Modules 1-3 covered the live machinery (how a prompt flows, who's involved, how tools work). This module is about **persistence**: how pi remembers conversations across restarts, branches, and forks.

**Course title:** Inside Pi: How a Coding Agent Actually Works
**Accent color:** vermillion
**Background:** Module 4 is even → `--color-bg`

### Screens (5 recommended)

1. **The conversation is on disk** — Open with the fact: every time pi runs, it writes a `.jsonl` file in `~/.pi/agent/sessions/`. Every user message, every model response, every tool call is appended as a single JSON line. Show a real path: `~/.pi/agent/sessions/-home-user-myproject/2025-05-16_<uuid>.jsonl`.

2. **The shape of one entry** — Show two or three sample JSONL lines (header + one user message + one assistant message). The key fields: `type`, `id`, `parentId`, `timestamp`, `message`. Code↔English translation. The big idea: `parentId` links each entry to the previous one, like a linked list — but because anyone can point to any parent, you get a **tree**, not a line.

3. **From list to tree** — Visualize. Start with a linear chain (msg1 → msg2 → msg3) as a horizontal flow. Then introduce a branch: a new entry whose `parentId` is `msg2`, creating a fork. Diagram:
   ```
   msg1 ── msg2 ──┬── msg3 ── msg4  (current path)
                  └── msg3b ── msg4b (alternate branch)
   ```
   Both paths live in the same file. The "current leaf" is wherever the user is right now.

4. **Walking the tree to build context** — When pi calls the model, it doesn't send the whole file. It **walks from the current leaf back to the root**, collecting all ancestors in order. That ordered list is the conversation the model sees. This is why branches don't pollute each other — only the ancestor chain matters.

5. **Branching in practice** — Three real commands:
   - `/tree` — Navigate the tree visually, jump to any past entry, continue from there. Old branch isn't deleted; the new path becomes the active one.
   - `/fork` — Copy the path from session start up to a chosen point into a brand-new session file.
   - `/clone` — Duplicate the current active branch as a new session.
   Use pattern cards or a flow diagram.

6. **Quiz**

### Code Snippets (pre-extracted)

**Snippet A — The base entry interface (conceptual, derived from `packages/coding-agent/src/core/session-manager.ts`):**
```
export interface SessionEntryBase {
    type: string;
    id: string;
    parentId: string | null;
    timestamp: string;
}

export interface SessionMessageEntry extends SessionEntryBase {
    type: "message";
    message: AgentMessage;
}
```

**Snippet B — Three real JSONL lines from a session file (illustrative, matching the documented format):**
```
{"type":"session","version":3,"id":"e3a1...","timestamp":"2025-05-16T14:00:00Z","cwd":"/home/me/proj"}
{"type":"message","id":"msg-1","parentId":null,"timestamp":"2025-05-16T14:00:01Z","message":{"role":"user","content":"list .ts files in src","timestamp":1747400401000}}
{"type":"message","id":"msg-2","parentId":"msg-1","timestamp":"2025-05-16T14:00:03Z","message":{"role":"assistant","content":[{"type":"text","text":"There are three: foo.ts, bar.ts, baz.ts."}],"model":"claude-sonnet-4-5","stopReason":"stop"}}
```

**Snippet C — Branching tree diagram (from `packages/coding-agent/docs/session-format.md`, slightly adapted):**
```
[user] ── [assistant] ── [user] ── [assistant] ─┬─ [user] ← current leaf
                                                 │
                                                 └─ [branch_summary] ── [user] ← alternate branch
```

### Interactive Elements

- [x] **Code↔English translation (Snippet A)** — Translate each field:
  - `type` → "What kind of entry this is. 'message' is the most common; others include 'compaction', 'branch_summary'."
  - `id` → "A short unique label for this entry, so other entries can point to it."
  - `parentId` → "The id of the entry that came right before this one in the chain. If null, this is the first entry."
  - `timestamp` → "When this entry was written."
  - For `SessionMessageEntry`: "A specific kind of entry — one that contains an actual conversation message. The `message` field holds the user/assistant/tool content."

- [x] **Code↔English translation (Snippet B)** — Walk the three lines:
  - Line 1 (the header): "Tells pi this is a v3 session, gives it an id, records when it started and what folder you were in."
  - Line 2 (user message): "The first message. Its parentId is null because nothing comes before it. The user said: 'list .ts files in src.'"
  - Line 3 (assistant message): "The model's reply. Its parentId is `msg-1`, linking it to the user's question. The reply text is in `content`. Pi also records which model produced this and why it stopped talking."

- [x] **Group Chat animation OR Visual file tree (REQUIRED — at least one)** — Recommended here: **Visual file tree** showing the tree structure with branches. Or use the **flow-steps** pattern with arrows to draw the parent links between three entries, then add a branch.

- [x] **Layer Toggle Demo (optional but great here)** — Tabs: "Linear view (what you see)" → "JSONL view (what's on disk)" → "Tree view (when you branch)." Each tab shows the same conversation rendered differently. This is a perfect fit for this module.

- [x] **Pattern cards** — Three cards for `/tree`, `/fork`, `/clone`. Each: icon, title, one-sentence description, example "when to use it":
  - **/tree** — "Navigate the tree in place. Jump to any earlier point and continue. Original branch is preserved."
  - **/fork** — "Copy the conversation up to a chosen point into a new file. Useful when you want to share or try something destructive."
  - **/clone** — "Duplicate the current branch as a new session. Useful for experimenting without losing the original."

- [x] **Quiz** — 3-4 scenario questions. Suggestions:
  1. "Pi just spent 20 turns refactoring your code, and the last 3 turns made everything worse. You'd like to keep most of the work but undo the last 3 turns. What do you do?" (Correct: use `/tree` to navigate back 3 entries and continue from there. The bad turns aren't deleted — they're just on a sidelined branch.)
  2. "You ran pi, closed the terminal by accident, and reopened it. Is your last conversation gone?" (Correct: no — it's in a JSONL file under `~/.pi/agent/sessions/`. Use `pi -c` to continue the most recent, or `pi -r` to browse and pick.)
  3. "Two team members opened the same session file. Can they both work on it at once?" (Correct: not safely — pi appends to the file, so concurrent writes will collide. Use `/clone` if they need parallel exploration.)
  4. "You want to compare two different approaches to a problem side by side. Which command sets that up best?" (Correct: `/fork` — branch from the user message where the two paths diverge, into separate session files. Run pi in two terminals, one per file.)

- [x] **Callout (aha!)** — "**A tree is just a linked list where each node can have multiple children.** That tiny change — letting parents point to multiple kids — unlocks branching, versioning, and undo for free. Git uses the same idea. So does your browser's history."

### Glossary Tooltips

- **JSONL** — JSON Lines. A file format where each line is one JSON object. Easy to append to (just add a new line) and easy to read one record at a time.
- **disk** / **on disk** — Saved to the computer's permanent storage (the hard drive or SSD). Survives shutdowns. Opposite of "in memory" which is lost when the program ends.
- **append** — Add to the end of a file without changing what's already in it. Pi never edits old session entries; it only appends new ones.
- **tree** (data structure) — A way of organizing data where each item has one parent and can have many children. Folders on your computer are a tree. Family trees are trees. Pi sessions are trees.
- **leaf** (tree) — A node with no children. The "current leaf" of a session is the most recent entry on the active branch.
- **root** (tree) — The topmost node. In pi sessions, the first user message is the root.
- **branch** — A path through a tree. When two children share a parent, each one starts a new branch.
- **fork** — Make a copy of something so you can change one without affecting the other. Used as a verb (to fork) and a noun (a fork).
- **linked list** — A simple data structure where each item points to the next one. A tree is a linked list where items can point to multiple "next" items.
- **uuid** — Universally Unique Identifier. A long random string used as a label, guaranteed to be unique. Session files use one in their name.
- **timestamp** — A label saying when something happened, usually in ISO format like `2025-05-16T14:00:01Z`.
- **stopReason** — Why the model stopped writing. Common values: `stop` (it was done), `length` (it ran out of token budget), `toolUse` (it wants to call a tool next).

### Reference Files to Read

- `references/content-philosophy.md` — required, full
- `references/gotchas.md` — required, full
- `references/interactive-elements.md` — sections: "Code ↔ English Translation Blocks", "Multiple-Choice Quizzes", "Visual File Tree", "Layer Toggle Demo", "Pattern/Feature Cards", "Flow Diagrams", "Callout Boxes", "Glossary Tooltips"
- `references/design-system.md` — Color Palette

### Connections

- **Previous module:** Module 3 — "Tools: The Model's Hands" — explained how the model affects the world. This module explains how all those actions get remembered.
- **Next module:** Module 5 — "When Context Runs Dry" — covers compaction. Long sessions can't fit in the model's memory; pi summarizes old turns to make room. Tease: "Long trees grow fast. The model has a maximum amount it can remember at once. What happens when you blow past it?"
- **Tone/style notes:** Vermillion accent; even module → `--color-bg`. Don't reuse "switchboard" / "film crew" / "DJ" metaphors. Choose-your-own-adventure / family tree is fresh.
