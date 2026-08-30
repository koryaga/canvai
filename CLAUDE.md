# canvai — DOM agent

A language model that lives inside one HTML page. The human writes on a
`contenteditable` canvas and leaves margin notes; the model answers by mutating
that same canvas. There is no chat window. **The agent's answer is the state of
the page, not a block of text.**

The hypothesis under test: a structured DOM store lets a model organise data for
human interaction better than a flat file tree does.

`ARCHITECTURE-min_1.md` is the binding spec. This file is the working summary —
where they disagree, the spec wins.

## Non-negotiables

**One file.** Everything is `min.html`, opened by double-click. No build step, no
server, no dependencies at runtime, no extra `.js` or `.css`. If a change needs
any of those, it is the wrong change.

**Two invariants.**

1. The diff of the human's edits is the only automatic channel of perception. A
   snapshot of the page is *never* sent to the model. If it needs state, it reads
   it itself with `page_exec`.
2. The scaffolding does not belong to the model. `#io`, `#debug`, the core and
   the handlers are outside its remit. Damage is detected by `checkFrame()` and
   cured by reload.

**Language.** Everything in the repository is English: code, comments, docs, the
system prompt, UI labels, log prefixes, turn sections. The only Russian is what
the human types on the canvas.

**Secrets.** `min.html` is tracked and ships with `API_KEY = ""`. Paste your key
straight into it to run, and blank it before committing. A `pre-commit` hook
rejects any `sk-…` in the index — it exists because a key was once committed. Do
not disable it. There is no second copy of the file and no other place a key
belongs.

## Page topology

| id       | visibility | role |
| -------- | ---------- | ---- |
| `#app`   | visible    | the canvas, `contenteditable`. Both the human's input and the model's output |
| `#store` | `hidden`   | the model's data layer, inert |
| `#io`    | visible    | status, pending counter, send and stop buttons |
| `#debug` | visible    | `<details open>`, the log: turns, calls, errors, final text |

There is no input field. The canvas is the only way in. Inside `#app` lives one
transient class of node: `<aside data-ask="#anchor">` — a margin note, not part
of the document.

## Script layout — the order is load-bearing

```
0. Config       constants
1. Scaffolding  element refs, log(), updateCount()
2. Observer     execWindow, buf/adds/dels, inAsk, raw, pathOf, process, drain, field listeners
3. Notes        ensureId, topLevel, addAsk, collectAsks
4. Diff         clip, diffLines, takeDiff, turnMessage
5. Execution    serialize, cut, errText, exec, runTool, checkFrame
6. Loop         SYSTEM, TOOLS, messages, setBusy, callModel, closeBatch, runTurn, commit
7. Bindings     contextmenu, keydown, Ctrl+Enter, buttons
```

Blocks call downwards — block 2 calls `clip` from block 4 — and that works only
through function-declaration hoisting. **Never convert a top-level `function`
into a `const` arrow, and never reorder the blocks.**

## The five mechanisms you must not break

These are listed in the spec as non-removable. Each failure mode below presents
as "the model is being stupid", never as an error.

**Execution window.** `execWindow` is raised around every DOM mutation the
*system* performs and lowered in `finally`; `process()` discards records that
arrive while it is up. Attribution is by window, never by `isTrusted` — the
model writes through `innerHTML`, which fires no events at all.

*Every system mutation site must be inside the window.* Today those are
`runTool`, `addAsk` and `setBusy`. Adding a sixth site without the window means
the system's own edit is reported to the model as the human's.

`addAsk` and `setBusy` save and restore the previous flag value rather than
forcing `false`: they can be invoked mid-turn while the window is already up.

**`drain()` routes through `process()`.** `takeRecords()` *clears* the queue.
Calling it and discarding the result silently loses everything the human did in
the last tick. The same `drain()` keeps records at commit time (window down) and
discards them at window close (window up).

**Coalescing.** On a repeat key the *old* value is kept from the first record and
the new value is read once, at diff assembly — `getAttribute()` during record
handling already returns the new value. Without this the diff reads «X → X».
`characterData` entries are keyed by the **text node object**, not by path:
`pathOf` addresses a text node by its parent, so sibling text nodes would collide
and all but the first edit would vanish.

**`raw` for comparison, `clip` for display.** Comparing clipped values makes two
strings that diverge past `CUT = 200` compare equal, and the human's edit
disappears with no diff line while the counter says one edit is pending.

**`takeDiff` is one synchronous block.** No `await` anywhere between draining
records and stripping the notes, or the human types in the gap and the text is
lost. Order: `drain()` (window down) → `diffLines()` → raise window →
`collectAsks()` + `reseat()` → `drain()` (window up) → lower window.

## Fix the contract, not the code

Most "the model behaves badly" reports are defects in `SYSTEM` (spec §8), and
they are a one-minute edit. Observed pattern worth remembering:

**A prohibition without a positive counterpart is read expansively.** The rule
"wrap interactive elements in `contenteditable=false`" made the model wrap whole
lists "to prevent editing the structure", killing the human's ability to edit
the document. Adding "the canvas is editable throughout and that is valuable —
do not take it away" fixed it immediately.

Same class: the prompt promised "returns the value your code returned" without
saying the code is an **async function body**. Code without `return` yields
`undefined`, which serialises to `"ok"` — so the model read the page, was told
`"ok"`, concluded the canvas was empty and re-emitted everything. It cannot
catch the lie, because of invariant 1.

## Transcript integrity

`messages` is module-scoped and outlives every turn. Any path that leaves an
`assistant` message's `tool_calls` without matching `tool` replies makes **every
later request in the session** invalid — surfacing as an HTTP 400 on a turn that
looks unrelated. `closeBatch()` exists for exactly this. Two guards follow from
it: never `return` mid-batch without closing it, and never push a message that
lacks a `role`.

Note the one deliberate deviation from the spec: the loop continues on a
non-empty `tool_calls` rather than on `finish_reason === "tool_calls"`, because
OpenAI-compatible providers frequently report `"stop"` alongside a populated
`tool_calls`. The disagreement is logged as `[provider]`. This still decides on a
protocol field, never by parsing response text — parsing ```js fences out of the
model's prose is forbidden.

## Out of scope — a prohibition list, not a backlog

`#units`, self-modification, storing the model's code; image persistence,
`localStorage`/`sessionStorage` in any form; emergency boot mode, a `booting`
watchdog, snapshots and rollback; isolation, a second origin, gates, sanitising
(security is out of scope); server, build, Web Workers, subagents, streaming;
external tools including outbound `fetch`; switching models from the UI; a
separate input field in any form, modal popups included.

Automatic rendering of the model's final text into `#app` is forbidden: it
collapses into chat mode and stops building structure, which is exactly what the
system exists to observe. Final text goes to `#debug`.

## Accepted risks

Reload wipes the session — canvas, `#store` and the transcript. Deliberate;
persistence is a later conversation. The transcript grows without trimming and a
long session will eventually hit a context-length 400. "Stop" aborts only the
network call, never synchronous page code. `Enter` produces `<div><br></div>`,
and `Backspace` merges nodes and carries ids away — the model learns of it only
when `querySelector` returns `null`.

## Working on this

**No test suite, by decision.** Verification is manual in a browser. Two
techniques replace it and are stricter than a live model run:

- `turnMessage(takeDiff())` returns exactly the string the model would receive.
  Call it directly; no API key needed. It is destructive — it drains the buffers
  and strips the notes.
- `await runTool("…")` simulates the model's edits through the identical path,
  execution window included. This is how you test attribution deterministically.

Before claiming a change works, exercise at least: coalescing keeps the *first*
old value; a model edit via `runTool` produces "nothing changed"; a note produces
INSTRUCTIONS and no DOCUMENT EDITS section.

**Verified live:** the default is OpenRouter with
`nvidia/nemotron-3-ultra-550b-a55b:free`. It and DeepSeek both work from an opaque origin —
`Origin: null` is accepted, so a `file://` page reaches them. Free-tier models do
support tool calling here. A generic `Failed to fetch` with no CORS message in
the console is a socket-level or extension problem, not an origin problem.
