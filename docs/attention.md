# Agent attention: "this agent is waiting for a human"

ReevesAgents answers one question the provider CLIs do not: **is any agent blocked on me
right now?** An agent that stops for an approval prompt, a clarify picker or a bare
question looks identical to an idle one in the run list — the tile says `working` forever
and the prompt itself sits inside a tmux window nobody is watching. This page describes
how that state is detected, stored and surfaced.

## What is detected

`src/core/attention.ts` classifies the last 40 lines of an agent's pane into one of three
reasons:

| reason | meaning |
|---|---|
| `approval` | a blocking approval / sudo modal, or a command refused because no prompt could be delivered |
| `input` | a clarify picker: the agent asked a question and waits for an answer |
| `question` | the turn finished and its last line is an unanswered question |

Everything else — working, finished with a statement, ended — reports `null`.

## Why pane text

No provider CLI exposes a machine-readable "waiting for a human" state; each renders its
own blocking overlay. The pane is therefore the only surface that is true for every
provider. Detection is a pure read of `tmux capture-pane` output: it never types into an
agent and never answers a prompt.

The modal is matched **structurally**, not by its wording, so it works in every UI
language Hermes ships (10 locales):

- a box corner (`╭` / `╰`),
- a numbered, selectable option line (`│ ❯ 1. …`),
- the `↑/↓` navigation hint,
- the answer countdown (`(38s)`).

Keywords only decide *which kind* of prompt it is: a `?` in the prompt body means a
question, approval words in ten languages mean an approval, and an unknown modal defaults
to `approval` because a decision is the safer assumption.

## Where it is stored

The sweep writes `attention` on the agent record in `~/.reeves/runs/<run> /agents/<id>.json`:

```json
"attention": { "reason": "approval", "detail": "rm -rf /tmp/build", "since": "2026-10-28T20:09:40.885Z" }
```

`since` is stamped once per prompt and preserved while the same prompt stays open, so a
tile can show how long the agent has been waiting. The field is cleared when the prompt
disappears. Writes happen only on change: a 2.5 s sweep over five agents does not churn
the registry.

The sweep runs in the web-UI/dashboard process (`startWebServer`), with or without a
browser attached — the point is to alert an operator who is **not** looking at that tab.
`attention` also shows up unchanged in the MCP `list` output and in the TUI, because both
read the same record.

## Where it is shown

- **Agent tile** (web UI): a filled pill with a pulsing dot — `approval needed`,
  `answer needed`, `question waiting` — in the warning colour, red for approvals. The
  tooltip carries `detail`, so the command or the question is readable without opening the
  window.
- **Tab title**: `(2) ReevesAgents web` while two agents need attention, which survives a
  background tab where the badge is not visible.
- **Pending MCP approvals** raised through `request_approval` are mapped onto the same
  badge (`source: "approval"`), so the two channels read the same way.

## Adding a provider

Detectors exist for `hermes` (approval and clarify modals) and `codex` (its command
approval prompt). Claude Code and the rest still report nothing. To add one:

1. Capture a real prompt: run the provider in a pane, trigger its approval overlay and
   `tmux capture-pane -p -S -40 -t <pane>`.
2. Write `detectXAttention(paneText)` returning `{ reason, detail } | null`, matching
   structure rather than wording whenever possible.
3. Register it in `DETECTORS` in `src/core/attention.ts`.

Nothing else changes: the sweep, the record field, the MCP output and the web badge are
provider-agnostic. Keep a captured pane as a fixture in `test/attention.test.ts` — when the
provider redesigns its overlay, that test is the alarm.
