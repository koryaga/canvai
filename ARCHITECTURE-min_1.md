# DOM agent — specification

The implementation is one file, `min.html`, opened by double-click. No build, no
server, no dependencies at runtime.

This document describes the whole system and is sufficient to implement it from
scratch. "Out of scope" is a prohibition, not a backlog. "Gotchas" is its most
valuable section: every entry there was paid for with a broken iteration.

---

## 1. Thesis and hypothesis

The agent lives inside an HTML page. The page is at once its working memory, its
state store, and its interface to the human. There are no external tools: the
model's only action is to run JavaScript inside this page.

There is no separate chat. The human writes directly on the canvas and the model
answers by changing that same canvas. **The agent's answer is the state of the
page, not a block of text.**

The hypothesis under test: a structured DOM store lets a model organise data for
interaction with a human more effectively than a flat file tree does.

## 2. Two invariants

1. **The diff of the human's edits is the only automatic channel of
   perception.** A snapshot of the page is never sent to the model. If it needs
   state, it reads it itself with `page_exec`.
2. **The scaffolding does not belong to the model.** `#io`, `#debug`, the core
   and the handlers are outside its remit. A violation is detected and cured by
   reload.

---

## 3. Page topology

| id       | visibility | role |
| -------- | ---------- | ---- |
| `#app`   | visible    | the canvas: `contenteditable="true"`, `spellcheck="false"`. Both the human's input and the model's output |
| `#store` | `hidden`   | the model's data layer, inert |
| `#io`    | visible    | status, pending counter, send and stop buttons |
| `#debug` | visible    | `<details open>`, the log: diff, calls, errors, final text |

There is no input field. The canvas is the only way in.

Inside `#app` lives one transient class of node: `<aside data-ask="#anchor">` —
notes (§5). They are not part of the document.

---

## 4. The observer

The subtlest part of the system. A mistake here does not look like a mistake —
it looks like "the model is being stupid".

### 4.1 Foundation — `MutationObserver`

```js
observer.observe(app, {
  childList: true, subtree: true, characterData: true,
  attributes: true, attributeOldValue: true, characterDataOldValue: true
});
```

An observer rather than input events, because **edits made through DevTools
produce no events at all**, and working through DevTools is a normal debugging
scenario.

### 4.2 Attribution — by execution window

The `execWindow` flag is raised for the duration of `page_exec` and lowered in
`finally`. Records that arrive inside the window are discarded: those are the
model's edits.

Attribution is **not by `isTrusted`**: the model's programmatic edits go through
`innerHTML`, not through events, and `isTrusted` says nothing about them.

Before closing the window — `observer.takeRecords()`, result to the bin: the
observer callback is asynchronous, and otherwise the model's records arrive after
the flag is lowered and get charged to the human.

Every DOM mutation the *system* performs belongs inside the window, not only
`page_exec`: today that is `runTool`, `addAsk` (§5) and the read-only toggle of
the canvas (§10). `addAsk` and the toggle save and restore the previous flag
value rather than forcing `false`, because they can run while the window is
already up.

### 4.3 Fields — separately

Live typing in an `<input>` changes the `.value` property, not an attribute. The
observer **never** sees it. Hence, delegated on `#app`:

- `focusin` — remembers the "before" value in `el.__was`;
- `input` and `change` — record `old → new`, only when `e.isTrusted`
  (programmatic assignment to `.value` fires no events, so the filter rejects
  only synthetic events from the model's code).

Checkboxes and radios are read through `.checked`, everything else through
`.value`.

### 4.4 Coalescing — mandatory

The buffer is a `Map`. On a repeat key **the old value is kept from the first
record**; the new one is not written at all — it is read once, at diff assembly.

The reason: `getAttribute()` at record-handling time already returns the current
value, not the one at mutation time. Without coalescing the diff reads «X → X».

The `Map` coalesces `characterData` and `attributes` only. For `childList` the
old state lives in the record itself, and coalescing it by path would merge
distinct added or removed nodes into one, so those go into plain arrays.

The key for `characterData` is **the node itself**, not the path. `pathOf`
addresses a text node by its parent, so two text children of one element —
`<p>start <b>b</b> end</p>` — would collapse into a single key, losing the second
edit without a trace. For attributes the path is unambiguous and there is no
collision.

**Raw** values are compared; `CUT` is applied for display only. Otherwise an edit
at the end of a paragraph longer than `CUT` makes the truncations compare equal,
no line is emitted at all, and the human loses text silently — while the pending
counter did show it.

### 4.5 Draining records

Record handling lives in a function `process(records)`; the observer is created
as `new MutationObserver(process)`, and `drain()` is
`process(observer.takeRecords())`.

Calling `observer.takeRecords()` and discarding the result is forbidden: edits
made in the same tick as the commit would be lost silently.

### 4.6 Node addressing

`pathOf(node)`: `#id` when present; otherwise a `tag:nth-of-type(n)` chain up to
the nearest ancestor with an `id`, prefixed `#app`. A text node is addressed by
its parent.

---

## 5. Notes: instruction versus document

The problem: the human writes both the document text and their requests to the
model on the same canvas. The model cannot tell them apart.

The solution is a margin comment, as in collaborative editors: the request to a
co-author lives next to the fragment, is visually distinct, and is not part of
the document.

### Creation

- right-click on the canvas (`Shift` + right-click leaves the browser menu);
- `Ctrl+/` — the same, by physical key (`e.code === "Slash"`): on a non-Latin
  layout `e.key` for that key is not `/`, and the human types on the canvas in
  their own language;
- `Escape` in an empty note cancels it. The handler is bound on `document`, not
  on `#app`, and finds the note by the caret: a freshly created empty `<aside>`
  leaves focus on `<body>`, so a listener on `#app` would never see the event.

`addAsk()`: takes `document.getSelection()`, walks up to the node that is a
direct child of `#app`, **assigns it an `id` of the form `n1` if it has none**
(without an anchor, "translate this" has no referent), inserts
`<aside data-ask="#n1" contenteditable="true">` after it, and focuses it.

### Collection

At commit time, inside `execWindow = true`: the text of every non-empty note goes
into the INSTRUCTIONS section and the node is removed from the page. The channel
is transient — its state does not survive the send. The removal does not enter
the diff.

### Exclusion from the diff

`process()` skips any record whose `target` lies inside `[data-ask]`: typing in a
note is not a document edit.

That is not enough. `addAsk()` puts an `id` on the anchor node — an attribute
edit **outside** `[data-ask]`, which would enter the diff. The model would be
told about an edit the human never made, on every first note to a node. So the
whole body of `addAsk` runs inside the execution window: assigning the `id`,
normalising bare text and inserting the `<aside>` are the system's work, not the
human's.

---

## 6. The turn message

Assembled in `takeDiff()`, which returns `{ asks, lines }`. Format:

```
INSTRUCTIONS (1):
[on #p1] translate this paragraph into English

DOCUMENT EDITS (3):
#p1 text: «one» → «one two»
#app + <li id="x">third</li>
#list − <li id="y"> «second»
#chk checkbox: «false» → «true»
```

Empty turn: `The human submitted a turn without changing anything.`

Limits: `MAX_DIFF = 60` records, `CUT = 200` characters per value, whitespace
collapsed everywhere.

`CUT` applies only to values in DOCUMENT EDITS — those are context, and
truncating them is harmless. It does not apply to INSTRUCTIONS text: an
instruction is not a value but the statement of the task itself, and a request
cut at character 200 reaches the model looking complete. Silently losing half the
task is worse than a long message.

Assembling the diff, stripping the notes and reseating the baseline (`el.__was`
across all fields) are **one synchronous block**. Otherwise the human types in
the gap and the text is lost.

---

## 7. The loop

```
human edits the canvas / leaves notes → Ctrl+Enter
  → takeDiff() atomically
  → turn message into the transcript
  → callModel
  → tool calls present ? exec → result into the transcript → repeat
                       : turn done
```

### Transport

`callModel` hits any OpenAI-compatible `/chat/completions`. The whole
configuration is three constants at the top of `min.html`, nothing else:

```js
const API_URL = "";   // OpenAI-compatible endpoint
const API_KEY = "";
const MODEL   = "";
```

`API_URL` and `MODEL` are committed filled in. The only secret is `API_KEY`, and
`min.html` **is tracked by git**: the key is pasted straight into it to run and
blanked before committing, with a pre-commit hook refusing any commit that
carries one. There is no second copy of the file — one file is the whole point.
Claiming "it does not leave" about a file that does leave is not acceptable: the
human reads that line at the exact moment they paste a paid key. There is no
storage either — the key is never saved and never read back.

The default is OpenRouter with `nvidia/nemotron-3-ultra-550b-a55b:free`.
Verified there and against DeepSeek (`deepseek-v4-flash`); both accept an opaque
origin, and free-tier models did support tool calling. A free endpoint is nonetheless a direct
candidate for the "free models and tool calling" gotcha in §10; switching models
on failure is a one-constant edit.

### The tool

Exactly one, native tool-calling. Parsing ```js fences out of the response text
is forbidden: it gives a second, unreliable execution criterion.

```json
{ "name": "page_exec",
  "parameters": { "type": "object",
    "properties": { "code": { "type": "string" } }, "required": ["code"] } }
```

`exec(code)` = `new Function("return (async () => {" + code + "})()")`.

The code runs as the **body of an async function**, not as an expression.
Without an explicit `return` the value is `undefined`, that is, `"ok"`. The
contract in §8 is obliged to say so: otherwise the model reads the page, gets
`"ok"`, and by invariant 1 cannot catch the lie — it concludes the canvas is
empty and re-emits what is already written.

Result serialisation: a string as is; `Node` → `outerHTML`; `NodeList` and arrays
of nodes → line by line; otherwise `JSON.stringify`; `undefined` → `"ok"`.
Truncated at `MAX_RESULT = 4000` with an explicit marker.

An exception is **not swallowed**: `name: message` plus three stack lines go to
the model as the tool result. This is the only self-repair mechanism.

Each stack line is capped at `CUT` **individually**, not just their count. A
frame can be arbitrarily long: on a page not opened from a file it carries the
entire source, and the model then gets `MAX_RESULT` characters of noise instead
of a diagnosis — along with everything the source contains, the key included.

### Stopping criterion

`finish_reason` is a protocol field, not the content of the response. A model
routinely emits text **and** a tool call in one message; a criterion of "it
answered with text" tears the turn on the first iteration. No separate `done()`
tool is introduced.

In practice the criterion is a non-empty `tool_calls` array rather than
`finish_reason === "tool_calls"`: OpenAI-compatible providers frequently report
`"stop"` alongside a populated `tool_calls`. The decision still rests on a
protocol field, never on parsing text, and the disagreement is logged.

Other conditions:

- `MAX_ITERS = 12`. At the ceiling, do not cut off silently: one more call with
  `tool_choice: "none"` and a message saying the limit is reached, wrap up.
- The stop button — `AbortController`.
- Three identical exceptions in a row (matching `name` + `message`) → the turn
  ends by force.
- A tool-argument parse error (cut off by `max_tokens`) is not a crash: the error
  text is returned to the model as the result.

### Transcript integrity

`messages` is module-scoped and outlives every turn. Any path that leaves an
`assistant` message's `tool_calls` without matching `tool` replies makes every
subsequent request of the session invalid — the provider sees `tool_calls` with
too few answers and returns 400 on a turn that looks unrelated. A turn that ends
mid-batch must therefore backfill the remaining calls, and no message without a
`role` may ever be pushed.

### Scaffolding

After each `page_exec` the scaffolding nodes are checked. On loss — `alert` and
`location.reload()`: the handlers are dead and there is nothing to repair from
the inside.

**All** scaffolding nodes are checked, not only the containers: `#log`, `#state`,
`#count`, `#send`, `#stop` alongside `#app`, `#store`, `#io`, `#debug`.
Otherwise `#debug.innerHTML = ""` leaves the container connected, the check
passes, and `log()` writes into a detached node — the human's only output channel
goes dark with no sign. Comparison is by node identity, not by presence of an
`id`, so replacement is caught too.

### Final text

Into `#debug`, not into `#app`. Rendering the answer into the canvas
automatically is forbidden: the model immediately collapses into chat mode and
stops building structure — losing exactly what the system is built to observe.

---

## 8. The contract with the model

The system prompt carries the page map, because the model gets no snapshot.
Mandatory points:

- the single tool `page_exec`, its returned value arrives as the result; the code
  runs as the body of an async function, and without `return` the result is
  `"ok"` rather than the value read;
- `#app` is the canvas, the answer to the human is what is written there;
  `#store` is data — keep state there, not in markup;
- a turn arrives in two sections: INSTRUCTIONS is the task anchored to a node;
  DOCUMENT EDITS is the new state of the text, context that usually needs no
  action;
- notes have already been stripped from the page; do not restore them;
- state survives the turn — do not re-emit what is written;
- put an `id` on nodes to be read later;
- `contenteditable="false"` goes on the interactive element itself and only on
  it; the attribute is inherited, and put on a container it takes away the
  human's ability to edit everything inside. The editability of `#app` is a value,
  not a side effect: without it the DOCUMENT EDITS section disappears, that is,
  half the loop. The contract is obliged to state this positively — otherwise the
  model silences whole lists "so the structure does not get broken", which was
  observed;
- text the human is meant to edit stays editable and carries an `id`;
- check `querySelector` against `null`;
- do not touch `#io`, `#debug`, or `body` as a whole.

A general lesson from both of the above: **a prohibition without a positive
counterpart is read expansively.** Most "the model behaves badly" reports are
defects in this section, and they cost a minute to fix.

---

## 9. Out of scope

Do not implement:

- `#units`, self-modification, storing the model's code;
- image persistence, `localStorage` in any form;
- an emergency boot mode, a `booting` watchdog, snapshots and rollback;
- isolation, a second origin, gates, sanitising — security is out of scope;
- a server, a build step, Web Workers, subagents, streaming;
- external tools, including outbound `fetch`;
- switching models from the UI;
- a separate input field in any form, modal popups included.

May not be cut: the atomicity of `takeDiff`, coalescing, the execution window,
`drain()`, returning the model's exception, the iteration ceiling, the
scaffolding check.

---

## 10. Gotchas

Each of these presents not as an error but as "the model works badly".

**`.value` is invisible to the DOM API.** Text typed into an `<input>` appears
neither in `textContent`, nor in `outerHTML`, nor to the mutation observer. Same
class: `checked`, `selected`, `scrollTop`, the openness of `<details>`. Each such
property needs its own listener.

**`takeRecords()` without handling loses edits.** The method returns **and
clears** the queue. Ignoring the return value means silently discarding
everything the human did in the last tick.

**The value at delivery time is already the current one.** Hence coalescing with
the "old" value fixed from the first record is mandatory.

**Async tails from the model.** Code in `page_exec` may leave a `setTimeout` or
`setInterval` behind; their mutations arrive after the window closes and get
charged to the human. Not curable in this architecture — accepted.

**Authorship race.** A human edit made during a turn falls inside the execution
window, is charged to the model, and is lost silently: the DOM changes while the
diff says "nothing changed". `MutationObserver` does not report authorship at
all, so there is nothing to tell them apart by.

Not curable, but preventable: for the duration of a turn `#app` is switched to
`contenteditable="false"` along with disabling the send button. Notes remain
available — `<aside>` carries its own `contenteditable="true"`, and its text is
read from the DOM at commit time rather than from the observer buffer, so it
survives the turn. The human cannot damage the document blindly, but can prepare
the next instruction.

**Buttons inside `contenteditable`.** A click places the caret instead of
pressing. Hence the requirement of `contenteditable="false"` on interactive
islands — and only on them, since the attribute is inherited.

**The caret litters the DOM.** `Enter` produces `<div>`/`<br>` to the browser's
taste, `Backspace` merges nodes and carries `id`s away. The model learns of it
only when `querySelector` returns `null`.

**Native undo is partial.** `Ctrl+Z` reverts the human's typing but not the
model's edits through `innerHTML` — those never enter the undo stack.

**`Origin: null` from `file://`.** A page opened by double-click sends exactly
that header in the preflight. The provider must accept it, or the request never
leaves: one CORS error in the console, zero calls in the log, and no answer from
the model. Checked on the very first turn. Verified as accepted by both providers
in §7 — a bare `Failed to fetch` with no CORS message in the console is a
socket-level or browser-extension problem, not an origin problem.

**Free models and tool calling.** Missing support looks like
`finish_reason: "stop"` with the code inside the response text and zero calls in
the log. Not to be confused with `429`/`503` — that is the provider's queue.

---

## 11. Acceptance

Manual scenarios:

1. "make a shopping list of three items" as a note → three `<li>` in `#app`.
2. The human edits an item's text by hand → `old → new` in the diff.
3. The human deletes an item → a removal line in the diff.
4. A turn where the model throws → the error text reached the model and it
   repaired itself within the same turn.
5. A note on a paragraph → an INSTRUCTIONS section with the anchor; the note is
   gone from the page; it does not appear in DOCUMENT EDITS.
6. A text edit plus a note in one turn → two sections in the message.
7. The model's own edits → absent from the next turn's diff.
8. `document.body.innerHTML = ''` during a turn → the page reloads.

There is no automated test suite in v1: acceptance is entirely manual. Two
techniques replace it and are stricter than a live run —
`turnMessage(takeDiff())` returns exactly the string the model would receive, and
`await runTool("…")` simulates a model edit through the identical path, execution
window included.
