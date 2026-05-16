# Module 6: Making Pi Yours

### Teaching Arc
- **Metaphor:** A power strip with empty outlets. Pi (the strip) is small on its own — a few minimal tools, a basic UI, the model loop. The interesting power comes from what you plug in: skills (instruction packs the model can use), extensions (TypeScript code that hooks into pi's events), prompt templates (saved shortcuts). Pi's philosophy is *to be the strip*, not the appliance.
- **Opening hook:** Other AI coding tools bake in features — plan mode, sub-agents, MCP, todo lists. Pi has none of these by default. That's intentional. Anything you'd want to add as a feature is already buildable as an extension. The result: a tiny core that becomes whatever you need.
- **Key insight:** Pi exposes three customization surfaces:
  1. **Skills** — markdown files in `~/.pi/agent/skills/` that the model reads on demand
  2. **Prompt Templates** — markdown files in `~/.pi/agent/prompts/` that expand when you type `/templatename`
  3. **Extensions** — TypeScript modules that register tools/commands/event handlers
  These three together let you reshape pi without touching pi's code.
- **"Why should I care?":** This is where vibe coders gain superpowers. You don't have to live with the defaults. A 30-line extension can add a confirmation dialog before any `rm -rf` command. A 5-line skill can teach pi to follow your team's PR style. This module is the "now you can actually build on top of this" payoff for the whole course.

### Course-Wide Context

Module 6 of 6 — the final module. Modules 1-5 explained how pi works internally. This module zooms out to show pi's deliberate, extension-driven design philosophy, and gives the learner concrete ways to extend it themselves.

**Course title:** Inside Pi: How a Coding Agent Actually Works
**Accent color:** vermillion
**Background:** Module 6 is even → `--color-bg`

### Screens (6 recommended — this is the climactic module)

1. **The minimalist's bet** — Open with the philosophy quote (Snippet A). Pi could have shipped with plan mode, todo lists, sub-agents, MCP, custom permission gates. It didn't. Why? Because every team's workflow is different, and a built-in feature is one you can't fully change. Code↔English translation.

2. **The three customization surfaces** — Pattern cards / icon rows showing the trio:
   - **Skills** — Markdown files. The model reads them as needed. Best for: instructions, conventions, how-tos.
   - **Prompt Templates** — Markdown files with placeholders. Best for: repeated prompts ("review this PR for security issues").
   - **Extensions** — TypeScript code. Best for: anything dynamic — new tools, event hooks, custom UI, integrations.

3. **A real extension, end to end** — Show the **permission-gate.ts** snippet (it's 30 lines, complete, real, useful). Code↔English translation. The pattern: register a handler for `tool_call`, inspect the call, return `{ block: true, reason: "..." }` to stop it. This single file blocks `rm -rf`, `sudo`, and `chmod 777` until the user confirms in the TUI.

4. **The event bus** — How extensions stay isolated. Show the safe-handler snippet from event-bus.ts. The pattern: extensions subscribe to channels (like "tool_call"), pi emits events on those channels, and even if one extension crashes, others keep working. Code↔English translation. This is the same pattern as a postal sorting facility: many recipients, one stream of letters, each recipient only opens their own.

5. **A tiny custom tool** — Show the **hello.ts** snippet (the smallest possible tool). Tie back to module 3: an extension is just a way to install one of those tools at runtime. Brief code↔English (or reference the one from module 3 if appropriate).

6. **What you can build (cards)** — Pattern cards inspired by the examples folder:
   - **Permission gates** — confirm before destructive commands
   - **Git checkpointing** — auto-stash at every turn
   - **Slack notifier** — ping a channel when long tasks finish
   - **Custom compaction** — keep summaries in your team's preferred shape
   - **Status lines / footers** — show project info in the TUI
   - **Games while you wait** — yes, snake.ts and a doom-overlay exist in the examples folder
   These aren't theoretical — they all exist in `examples/extensions/`.

7. **Quiz + course wrap**

### Code Snippets (pre-extracted)

**Snippet A — The philosophy quote (from `packages/coding-agent/README.md` line 470):**
```
Pi is aggressively extensible so it doesn't have to dictate your workflow.
Features that other tools bake in can be built with extensions, skills, or
installed from third-party pi packages. This keeps the core minimal while
letting you shape pi to fit how you work.
```

**Snippet B — The full permission-gate extension (from `packages/coding-agent/examples/extensions/permission-gate.ts` lines 1-34):**
```
/**
 * Permission Gate Extension
 *
 * Prompts for confirmation before running potentially dangerous bash commands.
 * Patterns checked: rm -rf, sudo, chmod/chown 777
 */

import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
    const dangerousPatterns = [/\brm\s+(-rf?|--recursive)/i, /\bsudo\b/i, /\b(chmod|chown)\b.*777/i];

    pi.on("tool_call", async (event, ctx) => {
        if (event.toolName !== "bash") return undefined;

        const command = event.input.command as string;
        const isDangerous = dangerousPatterns.some((p) => p.test(command));

        if (isDangerous) {
            if (!ctx.hasUI) {
                // In non-interactive mode, block by default
                return { block: true, reason: "Dangerous command blocked (no UI for confirmation)" };
            }

            const choice = await ctx.ui.select(`⚠️ Dangerous command:\n\n  ${command}\n\nAllow?`, ["Yes", "No"]);

            if (choice !== "Yes") {
                return { block: true, reason: "Blocked by user" };
            }
        }

        return undefined;
    });
}
```

**Snippet C — The hello tool (from `packages/coding-agent/examples/extensions/hello.ts` lines 8-26):**
```
const helloTool = defineTool({
    name: "hello",
    label: "Hello",
    description: "A simple greeting tool",
    parameters: Type.Object({
        name: Type.String({ description: "Name to greet" }),
    }),

    async execute(_toolCallId, params, _signal, _onUpdate, _ctx) {
        return {
            content: [{ type: "text", text: `Hello, ${params.name}!` }],
            details: { greeted: params.name },
        };
    },
});

export default function (pi: ExtensionAPI) {
    pi.registerTool(helloTool);
}
```

**Snippet D — Event bus safe handler (from `packages/coding-agent/src/core/event-bus.ts` lines 18-28):**
```
on: (channel, handler) => {
    const safeHandler = async (data: unknown) => {
        try { await handler(data); }
        catch (err) { console.error(`Event handler error (${channel}):`, err); }
    };
    emitter.on(channel, safeHandler);
    return () => emitter.off(channel, safeHandler);
},
```

### Interactive Elements

- [x] **Code↔English translation (Snippet A — philosophy)** — "Pi makes one strong bet: instead of building every workflow into the core, ship a small core and give users a way to extend it. The same way Unix shipped tiny tools and let you pipe them together. This is the lens for everything in this module."

- [x] **Code↔English translation (Snippet B — permission-gate)** — Walk through each piece. Highlight:
  - `export default function (pi: ExtensionAPI)` → "Every extension is a function that pi calls at startup, handing you the extension API as `pi`."
  - `const dangerousPatterns = [...]` → "A list of regular expressions matching commands we want to gate."
  - `pi.on("tool_call", async (event, ctx) => {...})` → "Subscribe to the `tool_call` event. Pi will run this function every time the model wants to call any tool."
  - `if (event.toolName !== "bash") return undefined` → "Only care about bash calls. For everything else, return `undefined` which means 'no opinion, let it through.'"
  - `if (isDangerous) { ... }` → "Check the command against our patterns."
  - `const choice = await ctx.ui.select(...)` → "Pop up a dialog in the TUI asking the user to allow or block."
  - `return { block: true, reason: "..." }` → "If they said no, return a block decision. Pi will not run the tool and will tell the model why."

- [x] **Code↔English translation (Snippet C — hello tool)** — Brief. Reference module 3 if needed. Highlight: this same tool definition shape works whether it's built into pi or installed via extension. **Same shape, different source.**

- [x] **Code↔English translation (Snippet D — event bus)** — "When an extension subscribes to an event, pi doesn't call the extension's handler directly — it wraps it in a try/catch. If the extension's code throws an error, pi logs it and keeps going. One bad extension can't crash pi. This is **isolation**."

- [x] **Pattern cards (REQUIRED — illustrates extensibility)** — The "what you can build" cards listed above. At least 6 cards.

- [x] **Group Chat animation (REQUIRED — at least one across the course, fits well here)** — Title: "How an extension intercepts a bash call." Actors:
  1. **Model**
  2. **Pi**
  3. **Permission Gate Extension**
  4. **You** (the user)
  Messages (no apostrophes):
  - *Model → Pi:* "Call bash with: `rm -rf node_modules`"
  - *Pi → Permission Gate:* "Heads up — model wants to call bash. Any objections?"
  - *Permission Gate:* "Checking the command against my danger patterns... rm -rf detected."
  - *Permission Gate → You:* "⚠️ Allow `rm -rf node_modules`? [Yes / No]"
  - *You → Permission Gate:* "No."
  - *Permission Gate → Pi:* "block: true. Reason: Blocked by user."
  - *Pi → Model:* "Tool blocked: user denied permission. Try a different approach."
  - *Model:* "Understood. I will ask the user before destructive commands from now on."

- [x] **Layer toggle OR side-by-side comparison** — Tabs: "Pi without extensions" → "Pi with permission gate" → "Pi with permission gate + git checkpoint + slack notify." Each tab shows what the same `rm -rf` command produces. Demonstrates that extensions **compose** — one event handler runs, then the next, etc.

- [x] **Callout (aha!)** — "**Pi is a thin layer that lets you treat AI coding tooling like Lego.** Every annoying thing — confirmation popups, todo lists, status bars, sub-agents — is a feature you can install or skip. The 'right' setup is the one you build."

- [x] **Quiz** — 3-4 scenario questions. Suggestions:
  1. "Your team uses a private Postgres database and you'd like pi to be able to query it. What's the cleanest path?" (Correct: write an extension that registers a `query-db` tool. Now the model can call it like any other tool. Don't try to teach the model SQL inline every time — make it a tool.)
  2. "You want every pi session, across your team, to read a `STYLE.md` file with your code conventions and follow them. Skill, template, or extension?" (Correct: **skill**. Skills are markdown the model reads on demand. Put STYLE.md as a skill, set it to be auto-discoverable, and the model will use it when relevant. Extensions are overkill for static instructions.)
  3. "You have a saved prompt 'review the diff and look for: $1.' You want to type `/review security` and have pi expand it. Skill, template, or extension?" (Correct: **prompt template**. Templates are exactly this — parameterized markdown, invoked by name with arguments.)
  4. "You wrote an extension that crashes on a specific event. Will it bring down the whole pi session?" (Correct: no — the event bus wraps every handler in a try/catch. Your extension's bug is logged but doesn't crash pi. This is why pi can safely run third-party extensions.)
  5. "You want to add a confirmation dialog before any tool that touches files in `~/secrets/`. Where do you implement that?" (Correct: an extension that hooks `tool_call`, inspects the args of `read`/`write`/`edit`/`bash` for the secrets path, and prompts the user via `ctx.ui.confirm()`. Exactly like permission-gate.ts.)

- [x] **Course wrap callout** — Final screen. Brief: "You started this course not knowing what a harness was. You now know how pi routes a prompt, who's in the cast, how the model touches the world, how memory branches like a tree, what compaction trades, and how to extend any of it. When you debug AI coding tools from here on, you have a map. Welcome to the inside."

### Glossary Tooltips

- **regular expression** / **regex** — A pattern for matching text. `\brm\s+(-rf?|--recursive)\b` matches the word "rm" followed by `-r`, `-rf`, or `--recursive`. Used in the permission-gate to spot dangerous commands.
- **subscribe** (events) — Tell a publisher "let me know when this event happens." Extensions subscribe to events; pi publishes them.
- **handler** — The function that runs when an event happens. The thing you pass to `pi.on("tool_call", handler)`.
- **isolation** — Keeping pieces of a program separate so one can't break the others. Pi's event bus wraps handlers in try/catch — that's isolation. Browsers run each tab in its own process — also isolation.
- **try/catch** — A programming pattern: "try this code, and if it errors, catch the error and do something safe instead of crashing."
- **MCP** — Model Context Protocol. A standard for letting AI tools talk to external services. Pi deliberately doesn't support it natively (you can add support with an extension); the maintainers think tool calls + READMEs are simpler.
- **sub-agent** — A separate AI session spawned by the main session to handle a sub-task. Pi doesn't bake this in; you can build it as an extension.
- **plan mode** — A feature in some AI tools where the model first writes a plan and asks you to approve before doing the work. Pi doesn't bake this in either — you can build it as an extension or save a "make a plan first" skill.
- **compose** (functions/extensions) — When several pieces stack on each other cleanly, each adding its layer without interfering. Pi extensions compose: install three, all three run.
- **Lego** (analogy) — A modular building system where every brick has standardized connectors. Pi treats itself like Lego — small, interchangeable pieces.
- **TUI** (if not in module 2 already) — Terminal User Interface — the text-based UI you see when running pi interactively.
- **TypeScript** (if not in module 2 already) — A flavor of JavaScript with type checking. Pi is written in TypeScript.

### Reference Files to Read

- `references/content-philosophy.md` — required, full
- `references/gotchas.md` — required, full
- `references/interactive-elements.md` — sections: "Code ↔ English Translation Blocks", "Multiple-Choice Quizzes", "Group Chat Animation", "Pattern/Feature Cards", "Callout Boxes", "Glossary Tooltips", "Layer Toggle Demo"
- `references/design-system.md` — Color Palette

### Connections

- **Previous module:** Module 5 — "When Context Runs Dry" — covered compaction. Mention briefly that compaction itself is hookable via an extension event (`session_before_compact`), tying compaction to extensibility.
- **Next module:** none — this is the final module. End with a wrap that explicitly references each module's takeaway and points toward the examples directory (`packages/coding-agent/examples/extensions/`) for further exploration.
- **Tone/style notes:** Vermillion accent; even module → `--color-bg`. Power-strip metaphor is unique to this module. Don't reuse switchboard, film crew, DJ, choose-your-own-adventure, or notebook metaphors.
