# Rendering Interactive-Session UX

The SSE/WebSocket patterns in `references/server-patterns.md` deliver the stream events; this file is about turning those events into UI — the live tool calls, diffs, plans, reasoning panels, and meters that make a custom browser frontend feel like a real Claude Code session.

## Table of Contents

- [The model: events are the contract, rendering is yours](#the-model-events-are-the-contract-rendering-is-yours)
- [Live tool-call entries](#live-tool-call-entries)
- [Streaming tool-input](#streaming-tool-input)
- [Reasoning vs answer](#reasoning-vs-answer)
- [Plan / todo display](#plan--todo-display)
- [File diffs](#file-diffs)
- [Context / cost meter](#context--cost-meter)
- [Attachments](#attachments)
- [Where these came from](#where-these-came-from)

---

## The model: events are the contract, rendering is yours

Every feature below maps to a stream event your server already forwards (or can
forward) from `claude -p --output-format stream-json --verbose
--include-partial-messages`. The server's job is to deliver those events
faithfully; the UI's job is to decide what each one *looks like*. The snippets
here use the same `esc()`-based, framework-light style as
`references/server-patterns.md#structured-result-rendering` — they assume a
helper like this is in scope:

```javascript
function esc(s) {
  const d = document.createElement('div');
  d.textContent = s == null ? '' : String(s);
  return d.innerHTML;
}
```

Nothing here requires React or a build step. Each section is a small function
that takes a parsed event and updates the DOM. Swap in your framework's
rendering primitives if you have one; the event handling is identical.

---

## Live tool-call entries

**Derived from:** `tool_use` blocks inside `assistant` events (the call: name +
input) paired with the later `tool_result` event (the outcome). The server
forwards `tool_use` blocks as they arrive (see the WebSocket pattern in
`references/server-patterns.md#pattern-websocket-session`, which already sends
`{ type: "tool", name, input }`); forward `tool_result` the same way.

A real session shows each tool call as its own entry — an icon, a one-line
summary of what it's doing, and a status that starts *running* and settles to
*done* or *failed* when the result lands. The one piece of plumbing that trips
people up: a `tool_result` carries **only** the `tool_use.id`, not the name or
input. Correlate by id. Keep a map from `id → entry` so the result can find the
entry it belongs to.

```javascript
// id -> { el, name } so a later tool_result can settle the matching entry
const toolEntries = new Map();

const TOOL_ICONS = {
  Read: '📖', Write: '✍️', Edit: '✏️',
  Glob: '🔍', Grep: '🔍', Bash: '⌨️',
  WebFetch: '🌐', TodoWrite: '☑️',
};

// Summarize the input differently per tool — the useful field varies.
function summarizeToolInput(name, input = {}) {
  switch (name) {
    case 'Read': case 'Write': case 'Edit':
      return input.file_path || '';
    case 'Glob': case 'Grep':
      return input.pattern || '';
    case 'Bash': {
      const cmd = input.command || '';
      return cmd.length > 60 ? cmd.slice(0, 60) + '…' : cmd;
    }
    default:
      // Fall back to the first short string field, or nothing.
      return '';
  }
}

// From an assistant event's tool_use block: render a running entry.
function renderToolCall(block) {
  const icon = TOOL_ICONS[block.name] || '🛠️';
  const summary = summarizeToolInput(block.name, block.input);
  const el = document.createElement('div');
  el.className = 'tool-entry tool-entry--running';
  el.innerHTML = `
    <span class="tool-icon">${esc(icon)}</span>
    <span class="tool-name">${esc(block.name)}</span>
    <span class="tool-summary">${esc(summary)}</span>
    <span class="tool-status">running…</span>`;
  document.getElementById('transcript').appendChild(el);
  toolEntries.set(block.id, { el, name: block.name });
}

// From a tool_result event: settle the matching entry by id.
function settleToolCall(result) {
  const entry = toolEntries.get(result.tool_use_id ?? result.id);
  if (!entry) return; // result for a call we never rendered — ignore
  const failed = result.is_error === true;
  entry.el.classList.remove('tool-entry--running');
  entry.el.classList.add(failed ? 'tool-entry--failed' : 'tool-entry--done');
  entry.el.querySelector('.tool-status').textContent = failed ? 'failed' : 'done';
}
```

Wire it into your event dispatch: on a forwarded `tool_use`, call
`renderToolCall`; on a forwarded `tool_result`, call `settleToolCall`. A call
that never receives a result stays *running* — which is honest, because the run
may have been interrupted before that tool finished.

---

## Streaming tool-input

**Derived from:** `stream_event` events of inner type `input_json_delta`
(available only with `--include-partial-messages`). Tool input does not arrive
all at once — it streams in as partial JSON fragments, the same way text streams
as `text_delta`. A long single `Write` can run for many seconds emitting only
input fragments and no text tokens at all, so without this the UI looks frozen.

Accumulate the fragments per tool-call index and show the entry "filling in" —
a byte counter is enough to prove progress ("Writing code… 12 KB"). The partial
JSON is usually not valid mid-stream, so do not try to `JSON.parse` it on every
delta; just measure it. Re-render only when the accumulated input actually
changed — fingerprint the buffer (length is a cheap fingerprint) and skip the
update if it is unchanged, so a burst of tiny deltas doesn't spam the DOM.

```javascript
// content-block index -> accumulated partial input + last rendered fingerprint
const partialInputs = new Map();

// event.event is the inner Anthropic stream event.
// content_block_start tells us the index belongs to a tool_use (and its name);
// input_json_delta carries the partial JSON for that index.
function onStreamEvent(inner) {
  if (inner.type === 'content_block_start' && inner.content_block?.type === 'tool_use') {
    partialInputs.set(inner.index, { name: inner.content_block.name, buf: '', fp: -1 });
    return;
  }
  if (inner.type === 'content_block_delta' && inner.delta?.type === 'input_json_delta') {
    const slot = partialInputs.get(inner.index);
    if (!slot) return;
    slot.buf += inner.delta.partial_json || '';
    // Fingerprint = byte length. Skip re-emit if unchanged to avoid event spam.
    if (slot.buf.length === slot.fp) return;
    slot.fp = slot.buf.length;
    showFilling(slot.name, slot.buf);
  }
}

function showFilling(name, buf) {
  const kb = (buf.length / 1024).toFixed(1);
  const label = name === 'Write' || name === 'Edit'
    ? `Writing code… ${kb} KB`
    : `Preparing ${esc(name)}… ${kb} KB`;
  const el = document.getElementById('active-tool');
  if (el) el.textContent = label;
}
```

When the matching `assistant` event arrives, its `tool_use` block carries the
*complete* input — promote the filling entry to a finished tool-call entry (the
previous section) and clear the partial slot for that index.

---

## Reasoning vs answer

**Derived from:** `content_block_delta` events. Extended-thinking models emit two
distinct delta types on the same channel: `delta.type === "thinking_delta"`
(text carried as `delta.thinking`) for the model's private reasoning, and
`delta.type === "text_delta"` (text carried as `delta.text`) for the answer the
user reads. The existing streaming code in
`references/server-patterns.md#pattern-sse-streaming` checks
`event.event?.delta?.text`, which naturally skips thinking deltas. To surface
reasoning, add a parallel branch on `event.event?.delta?.thinking` and route it
to a separate stream so the UI can show reasoning in a foldable panel,
visually distinct from the answer.

On the server, forward both as separate event types:

```typescript
const parse = createStreamParser((event) => {
  if (event.type === "stream_event" && event.event?.delta?.text) {
    send({ type: "token", text: event.event.delta.text });          // answer stream
  } else if (event.type === "stream_event" && event.event?.delta?.thinking) {
    send({ type: "thinking", text: event.event.delta.thinking });   // reasoning stream
  }
  // … tool_use / result handling as in the streaming patterns
});
```

On the client, keep two sinks. Reasoning goes into a collapsible panel that the
user can fold away; the answer renders inline as usual.

```javascript
function onMessageEvent(data) {
  if (data.type === 'thinking') {
    const panel = document.getElementById('reasoning-body');
    panel.textContent += data.text;
    document.getElementById('reasoning').hidden = false; // reveal the foldable panel
  } else if (data.type === 'token') {
    document.getElementById('answer').textContent += data.text;
  }
}
```

```html
<details id="reasoning" hidden>
  <summary>Reasoning</summary>
  <pre id="reasoning-body"></pre>
</details>
<div id="answer"></div>
```

Folding reasoning into a `<details>` keeps the answer prominent while still
letting a curious user inspect how Claude got there.

---

## Plan / todo display

**Derived from two sources.** (1) The `TodoWrite` tool — detect it in the
streaming tool-input (above) and render its `todos` array as a live checklist
that updates every time Claude rewrites it. (2) Plan mode — Claude signals a
finished plan by calling the `ExitPlanMode` tool with the plan as its input.

**Live todo checklist.** When a `TodoWrite` tool call finishes filling in, its
input is `{ todos: [{ content, status, activeForm }] }`. Re-render the whole
list on each `TodoWrite` — Claude sends the full current state every time, not a
diff.

```javascript
function renderTodos(input) {
  const todos = input.todos || [];
  const mark = { completed: '☑', in_progress: '◐', pending: '☐' };
  return `<ul class="todos">` + todos.map(t => `
    <li class="todo todo--${esc(t.status)}">
      <span class="todo-mark">${mark[t.status] || '☐'}</span>
      ${esc(t.status === 'in_progress' ? t.activeForm : t.content)}
    </li>`).join('') + `</ul>`;
}
```

**Plan mode in a GUI.** In a terminal, `ExitPlanMode` pauses and asks the user to
approve the plan before Claude executes. A browser app gets the same behavior by
intercepting the call at the approval gate rather than letting it auto-proceed.
Use the `PreToolUse` approval hook documented in
`references/advanced-patterns.md#http-hooks`: when the proposed tool is
`ExitPlanMode`, surface its `input.plan` as a card in the UI and **deny** the
tool with a message that tells Claude to wait for feedback. Denying (instead of
allowing) is what makes plan mode behave like a real plan-mode session in a GUI
— Claude stops and waits instead of charging ahead.

```typescript
// Inside the PreToolUse hook handler (see references/advanced-patterns.md#http-hooks).
// hookEvent.tool_name / hookEvent.tool_input come from the hook payload.
if (hookEvent.tool_name === "ExitPlanMode") {
  // Show the plan to the user as a card; collect their approve/reject out-of-band.
  broadcastToUI({ type: "plan", plan: hookEvent.tool_input.plan });
  return {
    decision: "block",   // deny the tool call
    reason: "Plan surfaced to the user. Wait for their feedback before executing.",
  };
}
```

The plan card itself is plain rendering — Markdown, or escaped text in a styled
panel with Approve / Revise buttons. Approve sends the next user turn ("proceed
with the plan"); Revise sends edits. The point is that the UI, not an
auto-proceed, decides when execution starts.

---

## File diffs

**Derived from:** `Write` / `Edit` `tool_use` calls and their `tool_result`s.
When your app edits files, showing the raw new contents is noise — show what
*changed*. The approach is the same regardless of diff library:

1. **Capture before/after.** For `Edit`, the tool input already contains
   `old_string` and `new_string`. For `Write`, read the file's prior contents
   before the write lands (or have the server emit a unified patch). Either a
   before/after pair or a unified patch is enough to render a diff.
2. **Render with a diff component + syntax highlighting.** Feed the patch (or the
   pair) to a diff renderer that produces line-level add/remove markup, and run
   the code through a highlighter for the file's language.
3. **Fall back to the raw patch on parse failure.** A malformed or truncated
   patch (mid-stream edits happen) should degrade to a monospace block of the
   raw patch text, never a blank panel or a thrown error.

Keep the renderer library-agnostic. `@pierre/diffs` is one option for the
diff-to-markup step, but anything that turns a unified diff into add/remove rows
works; the contract is the patch, not the library.

```javascript
// Library-agnostic shell. renderUnifiedDiff() is whatever diff component you choose.
function renderFileDiff({ path, patch }) {
  let body;
  try {
    body = renderUnifiedDiff(patch); // -> HTML with add/remove rows + highlighting
  } catch {
    // Truncated or malformed patch: show the raw text rather than nothing.
    body = `<pre class="diff-raw">${esc(patch)}</pre>`;
  }
  return `
    <div class="file-diff">
      <div class="file-diff__path">${esc(path)}</div>
      ${body}
    </div>`;
}
```

Optionally, accumulate per-file stats (`+adds / −dels`) across a turn and render
a **changed-files tree** — a sidebar list of touched paths with their line
counts, each expanding to the diff. This mirrors what a code editor shows after
an agent run and makes a multi-file turn legible at a glance.

---

## Context / cost meter

**Derived from:** the `result` event's `usage` object (and any
`rate_limit_event`s along the way — see the event table in
`references/server-patterns.md#stream-json-event-types`). The usage object
reports the tokens that count against the context window. Sum the input side
*including* cache tokens, because a cache read still occupies context:

```
used = input_tokens
     + cache_creation_input_tokens
     + cache_read_input_tokens
     + output_tokens
```

Show it as a context-window meter — used vs the model's max — so the user can
see how full the window is getting and when auto-compaction is near.

```javascript
function renderContextMeter(usage, contextWindow) {
  const used =
    (usage.input_tokens || 0) +
    (usage.cache_creation_input_tokens || 0) +
    (usage.cache_read_input_tokens || 0) +
    (usage.output_tokens || 0);
  const pct = Math.min(100, Math.round((used / contextWindow) * 100));
  const near = pct >= 80 ? ' meter--warn' : '';
  return `
    <div class="context-meter${near}" title="${used} / ${contextWindow} tokens">
      <div class="context-meter__fill" style="width:${pct}%"></div>
      <span class="context-meter__label">${pct}% of context used</span>
    </div>`;
}
```

When the stream reports a context compaction (a `compact` event, or
auto-compaction reflected in a later `result`'s usage dropping), surface that as
a small "compacted" status next to the meter so the user understands why the
window suddenly has room again.

**Do not present `total_cost_usd` as a bill.** It appears on the `result` event
even for subscription users and reflects internal accounting, not what anyone
is charged. If you show it at all, label it as an estimate of model usage, never
as money owed — surfacing it as a dollar amount misleads subscription users who
pay a flat rate.

---

## Attachments

**Derived from:** the user's own input, not a Claude event. When the user drops
or pastes images and PDFs, render them as a preview grid in the message you're
composing. The server resolves each attachment's temporary file path into the
prompt so Claude can Read it — see
`references/server-patterns.md#handling-file-uploads-and-drops` for the
upload/temp-path handling that backs this.

The UI side is a thumbnail grid built from the in-browser `File` objects (before
upload) or from server-returned preview URLs (after):

```javascript
function renderAttachmentGrid(files) {
  return `<div class="attachments">` + files.map(f => {
    const isImage = f.type.startsWith('image/');
    const isPdf = f.type === 'application/pdf';
    const thumb = isImage
      ? `<img class="att-thumb" src="${esc(f.previewUrl)}" alt="${esc(f.name)}">`
      : `<span class="att-thumb att-thumb--file">${isPdf ? '📄' : '📎'}</span>`;
    return `
      <figure class="attachment">
        ${thumb}
        <figcaption>${esc(f.name)}</figcaption>
      </figure>`;
  }).join('') + `</div>`;
}
```

For images, `f.previewUrl` is a `URL.createObjectURL(file)` you create on drop
(revoke it when the message is sent). PDFs and other Claude-readable binaries get
a generic file thumbnail — the point of the grid is to confirm to the user
*what they attached* before the turn runs, not to render the document.

---

## Where these came from

These rendering patterns are derived from t3code's
`apps/web/src/components/chat/` reference implementation, which renders a full
Claude Code session — tool calls, streaming input, reasoning, plans, diffs, and a
context meter — in a custom web UI. The shape of each stream event is the
contract; how you render it is the app's choice. Treat the snippets here as the
minimum that proves the event is wired up correctly, then style and elaborate to
fit your app.
