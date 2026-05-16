# Module 1: The Switchboard

### Teaching Arc
- **Metaphor:** A telephone switchboard operator from the 1950s — your prompt is a phone call that has to be physically routed through several junction boxes before it reaches the right person. The "operator" here is pi itself: it doesn't take the call, it routes it.
- **Opening hook:** You type "summarize this file" and hit Enter. From your point of view, the AI just answered. But between your Enter key and that answer, your message passed through at least five distinct stages — each of which can be intercepted, modified, or sabotaged.
- **Key insight:** Pi is not the AI. Pi is the **harness** around the AI — a piece of software that takes your text, routes it to a language model, gives the model some tools, runs whatever the model asks for, and brings the results back. Knowing the route is how you steer it.
- **"Why should I care?":** When AI gets stuck in a loop, hangs on a tool call, or "ignores" your instructions, you can almost always trace it to a specific stage in this route. Knowing the stages means you can ask sharp questions instead of vague ones.

### Course-Wide Context

This is module 1 of 6. The course teaches how the pi coding agent harness works. Pi is a terminal-based AI coding assistant similar to Claude Code. The user installs pi, runs `pi`, and types prompts to a language model that can read/write files and run shell commands. The whole course traces what happens between the user's keystrokes and the AI's response.

**Course title:** Inside Pi: How a Coding Agent Actually Works
**Accent color:** vermillion (warm red-orange)
**Background:** Module 1 is odd-numbered → use `--color-bg-warm` (alternating modules use `--color-bg` / `--color-bg-warm`).

### Screens (4-5 recommended)

1. **What pi actually is** — Open by explaining: pi is a terminal tool that wraps an LLM. You type, the model responds, pi handles the messy bits in between (reading files, running shells, keeping memory, etc.). Show the visual: terminal → pi (a translucent box) → model API. The model is OpenAI/Anthropic/etc.; pi is the harness. This grounds the whole course.

2. **The journey of one prompt** — Walk through what happens when the user types `summarize this README` and hits Enter. Five stages:
   - **Input capture** (interactive mode reads the editor, print mode reads argv)
   - **Preflight** (parse slash commands, expand skills, check if model is set up)
   - **System prompt assembly** (mix base instructions + tool descriptions + skills + project context)
   - **LLM call** (`agent.prompt(messages)`)
   - **The turn loop** (LLM asks for a tool → pi runs it → result back to LLM → repeat until LLM stops)
   Use a numbered step-card or flow diagram.

3. **The turn loop** — The single most important concept in the whole course. The model isn't a one-shot oracle. It works in **turns**: the model speaks, asks for a tool, pi runs the tool, the model speaks again, asks for another tool, pi runs that one. The loop continues until the model says "I'm done." Animate this. Use the data-flow animation.

4. **The four built-in tools** — Quick introduction: `read`, `write`, `edit`, `bash`. The model can only affect the world through these four. (Plus a few helpers: `find`, `grep`, `ls`.) Show a tiny code translation of the bash tool's schema so learners see what "a tool" actually is — a name, a description, and parameters.

5. **Quiz** — 3-4 questions testing understanding of the loop, not memory.

### Code Snippets (pre-extracted — use VERBATIM)

**Snippet A — What AgentSession is (from `packages/coding-agent/src/core/agent-session.ts` lines 1-14):**
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

**Snippet B — The bash tool schema (from `packages/coding-agent/src/core/tools/bash.ts` lines 23-26):**
```
const bashSchema = Type.Object({
    command: Type.String({ description: "Bash command to execute" }),
    timeout: Type.Optional(Type.Number({ description: "Timeout in seconds (optional, no default timeout)" })),
});
```

### Interactive Elements

- [x] **Code↔English translation (Snippet A)** — Show the AgentSession class comment. Translate each bullet into plain English: "This thing is the brain that orchestrates everything between your keyboard and the AI."
- [x] **Code↔English translation (Snippet B)** — The bash tool schema. Translate: "When the model wants to run a shell command, this is the *shape* of the request it has to send. Name it 'bash', include a command string, optionally a timeout."
- [x] **Data flow animation (REQUIRED)** — 5 actors in a row:
  1. **You** (terminal/keyboard icon)
  2. **Pi** (gear icon)
  3. **System Prompt** (scroll icon)
  4. **The Model** (brain icon)
  5. **Tools** (hammer icon, returns to Pi)
  Steps (write the data-steps JSON with **no apostrophes in labels**):
  - "You type a prompt and hit Enter" (highlight You)
  - "Pi receives the message and runs preflight checks" (highlight Pi, packet from You to Pi)
  - "Pi assembles the system prompt with tools and context" (highlight System Prompt, packet from Pi to System Prompt)
  - "The full conversation goes to the language model" (highlight Model, packet from Pi to Model)
  - "The model wants to run a tool" (highlight Model, packet from Model to Tools)
  - "Pi runs the tool and sends results back" (highlight Tools, packet from Tools to Model)
  - "The model decides it has enough info and writes the answer" (highlight Model)
  - "Pi delivers the answer to your terminal" (highlight You, packet from Model to You)
- [x] **Numbered step cards** — Five stages of the prompt journey (input → preflight → system prompt → LLM call → turn loop)
- [x] **Quiz** — 3-4 questions, scenario-style. Suggested questions:
  1. "You ask pi to read a file. The model 'thinks' for a moment, then says the file doesn't exist — but you can see it in your editor. Where does the bug most likely live?" (correct: the tool execution stage, because the file path got passed to the bash/read tool incorrectly. NOT the model — the model is just relaying what the tool told it.)
  2. "Pi seems to be in a back-and-forth loop running the same shell command over and over. What's happening?" (correct: the model is stuck in its turn loop — each turn it asks for the same tool because the result didn't satisfy it. Steer it with a new message or abort.)
  3. "You want pi to deploy to production. What has to be true for this to even be possible?" (correct: there has to be a tool — built-in or extension — that the model can call to do the deploy. The model can't 'deploy' directly; it can only describe a tool call.)
- [x] **At least one callout (aha! box)** — Suggested: "The model never touches your computer. It only writes text. Every file change, every command, every API call is pi running a tool on the model's behalf. If something happens, pi did it — not the AI."

### Glossary Tooltips (mark on first use)

Use the `.term` span with `data-definition` attribute. Mark these terms on first appearance:

- **LLM** / **language model** — A statistical text-prediction program trained on a huge amount of text. Given some text, it predicts what text should come next. That's it — but if you give it the right text, "what comes next" can be code, an answer, or a tool call.
- **harness** — The wrapper program around an AI model. The model just produces text; the harness reads that text, runs tools when the text describes a tool call, and feeds results back. Pi is a harness; ChatGPT.com is also a harness, just for chat instead of coding.
- **CLI** — Command-Line Interface. A program you control by typing text into a terminal window instead of clicking buttons. Pi is a CLI.
- **terminal** — The black window where you type commands like `cd` or `ls`. On Mac it's called Terminal; on Windows it's PowerShell or Windows Terminal.
- **tool** (in this context) — A function the AI model can ask the harness to run on its behalf. Pi ships four built-in tools: read, write, edit, bash.
- **system prompt** — A block of text the harness prepends to every conversation, invisible to the user. It tells the model what role it should play, what tools it has, what guidelines to follow.
- **API** — Application Programming Interface. The "phone number" of an internet service. Pi calls Anthropic's API to talk to Claude, or OpenAI's API to talk to GPT-4o.
- **turn** — One round-trip of the model: it speaks once (possibly asking for tools), pi handles whatever it asks for, and then the model speaks again. A single user prompt usually triggers multiple turns.
- **token** — Roughly a word or word-piece. LLMs measure their input/output in tokens. "Hello world" is about 2 tokens.
- **shell** / **bash** — The program that runs terminal commands. `bash` is one specific shell, but most people use the words interchangeably.
- **schema** — A description of what a piece of data must look like. The bash schema says: "this tool call must have a string named `command` and may have a number named `timeout`."

### Reference Files to Read

- `references/content-philosophy.md` — required, read in full
- `references/gotchas.md` — required, read in full
- `references/interactive-elements.md` — sections: "Code ↔ English Translation Blocks", "Multiple-Choice Quizzes", "Message Flow / Data Flow Animation", "Numbered Step Cards", "Callout Boxes", "Glossary Tooltips"
- `references/design-system.md` — Module Structure section + Color Palette section (you need to know the actor colors)

### Connections

- **Previous module:** none (this is module 1)
- **Next module:** Module 2 — "Meet the Cast" — introduces the major components (AgentSession, Tools, Extensions, SessionManager, Compaction, Modes, Skills) so the learner can identify which "character" does what. End this module by teasing: "We saw the route — next, let's meet the players who staff each junction box."
- **Tone/style notes:** Vermillion accent; warm, conversational; aimed at "vibe coders" who use AI coding tools but don't know what's under the hood. Avoid CS jargon without tooltips. Code snippets are real and must not be edited.
