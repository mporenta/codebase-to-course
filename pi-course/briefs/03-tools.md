# Module 3: Tools — The Model's Hands

### Teaching Arc
- **Metaphor:** A radio DJ inside a soundproof booth. The DJ can talk all day, but if they want to play a song, they have to press a labeled button on the panel in front of them — they cannot reach through the glass to spin a record themselves. The model is the DJ; tools are the buttons. The "panel" is the system prompt that lists which buttons exist.
- **Opening hook:** When AI "runs a command for you," it doesn't actually run anything. It writes a sentence that means "please run this command on my behalf" — and pi reads that sentence and runs it. The model is locked behind glass. Tools are the only buttons on its side.
- **Key insight:** Every action the model takes in the real world — opening a file, editing code, calling an API, running a build — must pass through a **tool call**. A tool is a named function with a schema; the model writes JSON that matches the schema; pi runs the corresponding code and feeds the result back.
- **"Why should I care?":** This explains so much that mystifies new users:
  - "Why can't the AI just deploy my app?" → Because there's no `deploy` tool. You'd have to give it one (extension), or describe a `bash` command it can run.
  - "Why does it sometimes hallucinate file contents?" → Because if it forgets to call the `read` tool, it makes the content up from training data. Always check that read was called.
  - "Why is it slow?" → Because every tool round-trip is a new API call. 10 tools used = 10 round-trips.

### Course-Wide Context

Module 3 of 6. Module 1 introduced the prompt journey. Module 2 named the cast. This module zooms into Tools — the most concrete, most "real" component. Once a learner understands tools, the rest of the course (sessions, compaction, extensions) is much easier.

**Course title:** Inside Pi: How a Coding Agent Actually Works
**Accent color:** vermillion
**Background:** Module 3 is odd → `--color-bg-warm`

### Screens (5-6 recommended)

1. **The DJ behind the glass** — Open with the metaphor. The model produces text; tools are how text becomes action. Visual: a brain icon (model) behind a glass pane, with labeled buttons (READ, WRITE, EDIT, BASH) on the user's side of the glass.

2. **Anatomy of a tool** — A tool is three things: a **name**, a **description** (so the model knows when to use it), and a **schema** (so the model knows what arguments to send). Show this concretely with the **hello tool** snippet — it's the smallest possible complete tool. Code↔English translation.

3. **The four built-in tools** — Pattern cards (one per tool):
   - **read** — Reads file contents. Handles giant files by truncating. Refuses binary files.
   - **write** — Creates or overwrites a file. Replaces the entire file.
   - **edit** — Modifies part of a file (a specific line range or text replacement). Keeps the rest intact.
   - **bash** — Runs a shell command. Streams output. Can be cancelled.
   Plus a sidebar mention: `find`, `grep`, `ls` (search/navigation helpers).

4. **What the model sees** — Show the bash tool schema snippet. Translate it: this exact JSON shape is what the model has to produce when it wants to run a command. The model doesn't "run" anything; it writes a JSON object, and pi reads that JSON and runs the matching code.

5. **The tool wrapper pattern** — Show the `wrapToolDefinition` snippet. This is a key engineering pattern: tools are described abstractly, but at execution time they need access to the session. The wrapper bridges that gap. Code↔English translation. Brief, but important for understanding extensions in module 6.

6. **Tool calls in the flow** — Animated data flow showing one tool round-trip:
   - Model emits a tool call (JSON)
   - Pi validates against schema
   - Pi runs the tool's code
   - Result formatted and sent back to model
   - Model continues

7. **Quiz**

### Code Snippets (pre-extracted)

**Snippet A — The hello tool, complete (from `packages/coding-agent/examples/extensions/hello.ts` lines 1-27):**
```
/**
 * Hello Tool - Minimal custom tool example
 */

import { Type } from "@earendil-works/pi-ai";
import { defineTool, type ExtensionAPI } from "@earendil-works/pi-coding-agent";

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

**Snippet B — Bash tool schema (from `packages/coding-agent/src/core/tools/bash.ts` lines 23-26):**
```
const bashSchema = Type.Object({
    command: Type.String({ description: "Bash command to execute" }),
    timeout: Type.Optional(Type.Number({ description: "Timeout in seconds (optional, no default timeout)" })),
});
```

**Snippet C — Tool definition wrapper (from `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts` lines 4-19, slightly excerpted):**
```
export function wrapToolDefinition<TDetails>(
    definition: ToolDefinition<any, TDetails>,
    ctxFactory?: () => ExtensionContext,
): AgentTool<any, TDetails> {
    return {
        name: definition.name,
        execute: (toolCallId, params, signal, onUpdate) =>
            definition.execute(toolCallId, params, signal, onUpdate, ctxFactory?.() as ExtensionContext),
    };
}
```

### Interactive Elements

- [x] **Code↔English translation (Snippet A — the hello tool)** — Walk through each piece:
  - `defineTool({...})` → "Build a tool. The thing inside the parens is its definition."
  - `name: "hello"` → "The model will refer to this tool by the word 'hello'."
  - `label: "Hello"` → "How the tool's name shows up in the pretty UI when it runs."
  - `description: "A simple greeting tool"` → "What the model reads to decide whether to use this tool. The model can only call tools it knows about, and it reads the description to know what each one does."
  - `parameters: Type.Object({ name: Type.String(...) })` → "The shape of arguments this tool expects. Here: one string argument called 'name'."
  - `async execute(...)` → "The actual code that runs when the model calls this tool."
  - `return { content: [...], details: {...} }` → "What the tool sends back. The 'content' goes to the model so it can keep working; 'details' is extra info the UI might show."
  - `pi.registerTool(helloTool)` → "Tell pi about this tool so it shows up in the system prompt and can be called."

- [x] **Code↔English translation (Snippet B — bash schema)** — "Define the *shape* of arguments. To call this tool, the model must send a JSON object with a string field named 'command'. Optionally, also a number named 'timeout'. If the model's JSON doesn't match this shape, pi rejects the call before it ever runs."

- [x] **Code↔English translation (Snippet C — wrapper)** — Brief. The "why": tools are written one way (with full context like "what session am I in?"), but agent-core expects a simpler shape. The wrapper acts as an **adapter** — it takes a fancy tool definition and exposes the simpler shape that agent-core wants, while keeping access to context.

- [x] **Pattern cards** — One card per built-in tool (read, write, edit, bash). Each card has an icon, title, one-sentence description, and a tiny example use ("Read README.md", "Write a config file", "Edit line 42", "Run npm test").

- [x] **Data flow animation (REQUIRED)** — Title: "One tool call, step by step." Actors:
  1. **Model** (brain icon)
  2. **Pi** (gear icon)
  3. **Tool: bash** (terminal icon)
  4. **The OS** (computer icon)
  Steps (NO apostrophes in labels):
  - "Model writes a tool call as JSON" (highlight Model)
  - "Pi receives the call and checks it against the schema" (highlight Pi, packet Model → Pi)
  - "Pi runs the bash tool with the supplied args" (highlight Tool, packet Pi → Tool)
  - "Bash tool launches a real shell process" (highlight OS, packet Tool → OS)
  - "Shell finishes; output comes back" (highlight Tool, packet OS → Tool)
  - "Pi packages the output as a tool result" (highlight Pi, packet Tool → Pi)
  - "The model reads the result and decides the next move" (highlight Model, packet Pi → Model)

- [x] **Spot-the-bug challenge** (optional, but a nice touch) — Show a fake tool call where the model produced bad JSON (e.g., missing `command` field). Ask which line is broken. Reveal: pi rejects the call before running, returns an error to the model, model tries again.

- [x] **Quiz** — 3-4 scenario questions:
  1. "You ask pi to 'delete my server'. Nothing happens — pi just says it can't do that. Why?" (Correct: there is no `delete-server` tool. The model can call `bash` to *try* a delete command, but if it doesn't have the right credentials or thinks the command is unsafe, it'll refuse. The model can't 'just do' anything outside its tools.)
  2. "Pi confidently tells you the contents of `package.json` but the contents it cites don't match what's actually on disk. What likely happened?" (Correct: the model skipped the `read` tool and **hallucinated** the contents from its training data. Look at the message log: if there's no read tool call, the contents are guessed. Steer pi by explicitly asking it to read the file first.)
  3. "You want pi to send a Slack message at the end of every long task. How would you set that up?" (Correct: write an **extension** that registers a new tool like `slack-notify`. Then prompt: 'when you're done, call slack-notify.' The model can now press that button.)
  4. "Pi just sat there for 30 seconds, then printed the contents of a file. What was it doing during the 30 seconds?" (Correct: it was inside the `read` tool, actually reading the file from disk. Tool execution time is not 'AI thinking' — it's pi running real code. Big files take real time.)

- [x] **Callout (aha!)** — "**The model is on the wrong side of the glass.** It speaks, but it doesn't act. If you want it to do something new — call an API, deploy a service, query a database — you (or an extension) have to install a tool that does that. Then the model can press the button."

### Glossary Tooltips

- **JSON** — JavaScript Object Notation. A way to write structured data as plain text. `{"name":"alice","age":30}` is JSON. The model and pi pass tool calls and results as JSON.
- **function** — A named chunk of code that takes some inputs and produces an output. A tool is a function the model can call.
- **schema** (if not in module 1) — A description of what valid data looks like. The bash schema says "must have a `command` string."
- **argument** / **parameter** — The inputs to a function. `add(2, 3)` calls the `add` function with arguments 2 and 3.
- **shell command** — A line of text you'd type into a terminal: `ls`, `git status`, `npm install`. Bash, zsh, and fish are all shells.
- **process** — A running program. When pi runs a bash command, it starts a new process to do the work.
- **adapter** (pattern) — A piece of code that converts one shape into another so two components can work together. Like an HDMI-to-USB-C dongle.
- **hallucinate** (AI context) — When an AI makes up information that sounds plausible but isn't real. If the model doesn't call the read tool, anything it "knows" about your file is hallucination.
- **abort signal** — A way to politely tell a running operation "stop now, the user pressed Cancel." Pi passes one of these into every tool so long operations can be interrupted.
- **stream** (output) — Data arriving in pieces over time, instead of all at once. Bash command output streams in line by line.
- **typebox** / **Type.Object** — A library pi uses to write schemas. `Type.Object({...})` is how pi says "expect a JSON object shaped like this."

### Reference Files to Read

- `references/content-philosophy.md` — required, full
- `references/gotchas.md` — required, full
- `references/interactive-elements.md` — sections: "Code ↔ English Translation Blocks", "Multiple-Choice Quizzes", "Pattern/Feature Cards", "Message Flow / Data Flow Animation", "Spot the Bug Challenge", "Callout Boxes", "Glossary Tooltips"
- `references/design-system.md` — Color Palette + Code Block Globals

### Connections

- **Previous module:** Module 2 — "Meet the Cast" — named all components. This module zooms into Tools, the most concrete and frequently-touched one.
- **Next module:** Module 4 — "Memory as a Tree" — covers how pi remembers what you and the model said. Tease: "Every tool call and every model message gets written down. The shape of that record is more interesting than you'd expect — it's a tree, not a list."
- **Tone/style notes:** Vermillion accent; odd module → `--color-bg-warm`. Keep code snippets verbatim from the codebase. Do not reuse "switchboard" or "film crew" metaphors from earlier modules. The DJ metaphor is unique to this module.
