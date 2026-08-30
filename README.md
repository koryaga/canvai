# canvai

A language model that lives inside one HTML page.

There is no chat window. You write on a `contenteditable` canvas and leave margin
notes; the model answers by changing that same canvas. Its only action is running
JavaScript in the page. **The answer is the state of the page, not a block of
text.**

One file, `min.html`. No build, no server, no dependencies.

## Run

Put your API key into `API_KEY` at the top of `min.html`, then open the file by
double-click. That is the whole setup — there is no second copy of the file.

`API_URL` and `MODEL` default to OpenRouter and a free model
(`nvidia/nemotron-3-ultra-550b-a55b:free`). Any OpenAI-compatible endpoint works;
switching is a one-constant edit.

The key lives in your working copy only. Blank it before committing — `min.html`
is tracked, and a pre-commit hook refuses any commit that carries a key.

## Use

- Type on the canvas — that is the document.
- Right-click (or `Ctrl+/`) to leave a note for the model, anchored to the
  paragraph you clicked. `Escape` cancels an empty one.
- `Ctrl+Enter` sends the turn: your notes as instructions, your edits as context.
- Watch the log at the bottom to see what the model ran.

The canvas goes read-only while the model works. You can still add notes for the
next turn.

## Read

`CLAUDE.md` — architecture principles and the things you must not break.
`ARCHITECTURE-min_1.md` — the full spec.

## Status

A working prototype. It has no test suite by design, verification is manual, and
a reload wipes the session.
