# Module 2: Meet the Cast

### Teaching Arc
- **Metaphor:** A film crew on a movie set. The model is the lead actor — it's what's on screen. But a movie also has a director (AgentSession) who decides what scene to shoot next, stunt doubles (Tools) who do the dangerous stuff, a script supervisor (SessionManager) who logs every take, a wardrobe department (Extensions) that can swap props mid-shoot, and three different cameras (Modes) that frame the same scene differently.
- **Opening hook:** When pi answers your prompt, it looks like one thing happened. It's actually seven things — and each one is a separate piece of code with a specific job. Once you can name them, you can debug them.
- **Key insight:** Pi is built around **separation of concerns**: each component does one thing well, and they communicate through clean boundaries. This is the most important pattern in software engineering. Knowing the cast lets you say "the model is fine, but the tool wrapper is wrong" instead of "the AI is broken."
- **"Why should I care?":** When you describe a bug to AI ("the file save isn't working"), you can be precise: "AgentSession got the tool result, but the edit tool returned an error." Precise descriptions get precise fixes. Vague descriptions get vague flailing.

### Course-Wide Context

Module 2 of 6. Module 1 introduced the journey of a prompt as a 5-stage route. This module zooms in on the **components** that staff each stage. Module 3 will dive deep into one of them (tools); this module is the wide shot.

**Course title:** Inside Pi: How a Coding Agent Actually Works
**Accent color:** vermillion
**Background:** Module 2 is even-numbered → use `--color-bg`

### Screens (5 recommended)

1. **Why split things into pieces?** — Open with the "separation of concerns" idea before naming any components. Use a metaphor: a restaurant has a chef, a server, a dishwasher — not because one person can't do all three, but because separating makes each job replaceable. If the dishwasher breaks, you can hire a new one without firing the chef. Pi works the same way. (Note: this is the ONLY restaurant reference allowed — use it briefly and move on, not as a recurring metaphor.) Actually, prefer this: think of an airport — separate teams handle check-in, security, baggage, fueling, air-traffic control. One plane, many specialists.

2. **The seven actors** — Use the **Icon-Label Rows** pattern (one row per actor). Each row has a colored icon + name + one-sentence description. Actors:
   - **AgentSession** (color: --color-actor-1) — The director. Coordinates everything: takes your prompt, calls the model, runs tools, persists results, decides when to compact.
   - **Agent** (color: --color-actor-2) — The messenger. The lower-level library that actually talks to the language model API. AgentSession wraps it.
   - **Tools** (color: --color-actor-3) — The hands. Functions the model can ask to run: read, write, edit, bash, plus search helpers.
   - **SessionManager** (color: --color-actor-4) — The script supervisor. Writes every message, tool call, and result to a `.jsonl` file on disk.
   - **Extensions** (color: --color-actor-5) — The wardrobe department. User-installed plugins that can register new tools, hook into events, add slash commands, or replace built-in behavior.
   - **Skills** + **Prompt Templates** (re-use --color-actor-3 or use a new color) — Markdown files that teach the model how to do specific tasks on demand.
   - **Modes** (re-use --color-actor-2) — The three cameras: interactive (the live TUI you see in the terminal), print (one-shot, scripting), RPC (machine-to-machine for IDE plugins).

3. **The architecture diagram** — Big interactive diagram showing how the actors connect. **Pi (core)** in the center, **Modes** wrapping around it (interactive/print/RPC are I/O layers), **AgentSession** as the orchestrator, branches out to **Agent → LLM** (external), **Tools**, **SessionManager → .jsonl files** (storage), **Extensions** hanging off the side as plug-ins, **Skills/Templates** feeding into the system prompt. Use the architecture diagram interactive element. Each clickable node shows a one-sentence description.

4. **AgentSession: the director** — Zoom in on AgentSession. Show the code↔English of its top comment. Explain: this class is the conductor. The other actors are tools and instruments it uses. Modes use AgentSession; they don't talk to the model directly.

5. **The three modes** — Three cards or icon rows showing interactive vs print vs RPC. Same engine, different presentation. Why this matters: you can use pi as a chat tool, a script command, or wire it into an IDE — without changing the core. This is **separation of concerns** paying off.

6. **Quiz** — 3-4 scenario questions.

### Code Snippets (pre-extracted)

**Snippet A — AgentSession header comment (from `packages/coding-agent/src/core/agent-session.ts` lines 1-14):**
```
/**
 * AgentSession - Core abstraction for agent lifecycle and session management.
 *
 * This class is shared between all run modes (interactive, print, rpc).
 * It encapsulates:
 * - Agent state access
 * - Event subscription with automatic session persistence
 * - Model and thinking level management
 * - Compaction (manual and auto)
 * - Bash execution
 * - Session switching and branching
 *
 * Modes use this class and add their own I/O layer on top.
 */
```

**Snippet B — The Skill schema (from `packages/coding-agent/src/core/skills.ts` lines 68-82, conceptual structure):**
```
export interface Skill {
    name: string;
    description: string;
    filePath: string;
    baseDir: string;
    sourceInfo: SourceInfo;
    disableModelInvocation: boolean;
}
```

**Snippet C — Pi's philosophy (from `packages/coding-agent/README.md` line 470):**
```
Pi is aggressively extensible so it doesn't have to dictate your workflow.
Features that other tools bake in can be built with extensions, skills, or
installed from third-party pi packages. This keeps the core minimal while
letting you shape pi to fit how you work.
```

### Interactive Elements

- [x] **Code↔English translation (Snippet A)** — Map each bullet of the AgentSession comment to one line of plain English. e.g., "Agent state access → Knows what's been said so far" / "Compaction (manual and auto) → Decides when to summarize the chat to save space" / "Modes use this class and add their own I/O layer on top → The three different ways to run pi all share this brain; only the screen they draw on differs."
- [x] **Code↔English translation (Snippet C)** — The philosophy quote. Translate: "Pi's main bet is that no harness can know every workflow in advance, so it ships small and lets you extend it. Other tools try to be one-size-fits-all; pi is a kit."
- [x] **Interactive architecture diagram (REQUIRED)** — Use the `.arch-diagram` pattern. Zones: "Your Terminal" (Modes), "Pi Core" (AgentSession, Agent, Tools), "Storage" (.jsonl session files), "Plug-ins" (Extensions, Skills, Templates), "External" (LLM API). Click any component to see a sentence describing its job.
- [x] **Icon-Label Rows** — One row per actor (the seven listed above).
- [x] **Group Chat animation (REQUIRED — at least one in this module)** — Title: "A typical turn, as a conversation." Actors: **AgentSession**, **Agent**, **The Model**, **Tools**. Messages (in order):
  1. *AgentSession → Agent*: "Here's the user's message and the full chat history. Ask the model what to do next."
  2. *Agent → Model*: "User says: 'list the .ts files in src'. What's your move?"
  3. *Model → Agent*: "Call the bash tool with `ls src/*.ts`."
  4. *Agent → AgentSession*: "Model wants to run bash. Here are the args."
  5. *AgentSession → Tools*: "Run this command. Here's the abort signal in case the user cancels."
  6. *Tools → AgentSession*: "Done. Output: foo.ts, bar.ts, baz.ts."
  7. *AgentSession → Agent*: "Send the result back to the model."
  8. *Agent → Model*: "Tool result: foo.ts, bar.ts, baz.ts. Continue."
  9. *Model → Agent*: "Three TypeScript files: foo, bar, baz. Done."
  Use distinct avatars/colors per actor. Avoid apostrophes in chat text (use straight or replace) to be safe.
- [x] **Quiz** — 3-4 scenario questions. Suggestions:
  1. "Pi is running fine in your terminal, but the same model 'works differently' when used from an IDE extension. What's likely different?" (Correct: the **mode** is different — IDE plugin uses RPC mode, terminal uses interactive. Same AgentSession, but the I/O layer is different.)
  2. "You want to log every tool call your team makes, across all developers, into a central file. Which component would you reach for first?" (Correct: an **extension** that hooks the `tool_call_event`. Not AgentSession itself — that's pi's internals. Extensions are the official extension point.)
  3. "The model keeps responding in a chatty, verbose style and you want terser answers across every session. Where does that personality live?" (Correct: the **system prompt** — and you can append to it with a setting or extension.)

- [x] **Callout (aha!)** — "**Separation of concerns** is one of the most important ideas in software. When each component owns one job, you can swap, debug, or scale any of them without disturbing the rest. It's why pi has three run modes built from one core."

### Glossary Tooltips

- **separation of concerns** — Splitting a program into pieces where each piece has exactly one job. When something breaks, you only have to look in one piece instead of the whole codebase.
- **orchestrator** — The piece that coordinates other pieces. It doesn't do the work itself; it tells other components when to do their work, in what order.
- **API** (if not already in module 1) — see module 1
- **plug-in** — A piece of code that hooks into a larger program to add features, without modifying the larger program. Browser ad-blockers are plug-ins for browsers; pi extensions are plug-ins for pi.
- **event** — A signal sent when something happens. "The user pressed a key." "A tool finished running." Extensions can listen for events and react.
- **RPC** — Remote Procedure Call. A way for two programs to talk: program A sends a structured message, program B does something and sends a structured message back. Like a function call, but across program boundaries.
- **TUI** — Terminal User Interface. A user interface drawn with text in a terminal window. Pi's interactive mode is a TUI.
- **TypeScript** — A flavor of JavaScript that adds type annotations so the computer can catch certain bugs before the program runs. Pi is written in TypeScript.
- **interface** (TypeScript context) — A description of the *shape* of an object: what fields it has, what types those fields are. Useful for catching mistakes before the program runs.

### Reference Files to Read

- `references/content-philosophy.md` — required, full
- `references/gotchas.md` — required, full
- `references/interactive-elements.md` — sections: "Code ↔ English Translation Blocks", "Multiple-Choice Quizzes", "Interactive Architecture Diagram", "Icon-Label Rows", "Group Chat Animation", "Callout Boxes", "Glossary Tooltips"
- `references/design-system.md` — Color Palette (actor colors)

### Connections

- **Previous module:** Module 1 — "The Switchboard" — covered the route a prompt takes. This module names the staff at each junction.
- **Next module:** Module 3 — "Tools: The Model's Hands" — drills into Tools specifically. Tease at the end: "We've named the cast. Next, let's follow the most important supporting actor — the tools — into the wings, and see how the model actually grabs them."
- **Tone/style notes:** Vermillion accent; even module → use `--color-bg` background. Don't reuse the "switchboard" metaphor from module 1.
