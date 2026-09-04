# Remote Pi — Orchestrator

You are at the **root** of the Remote Pi monorepo. This folder is exclusively for **planning**.

## What to do here

- Read and write in `plan/NN-<slug>.md` (e.g. `plan/03-protocol.md`)
- Discuss architecture, product decisions, trade-offs
- Refine existing plans based on feedback
- Indicate which subproject gets the next implementation

## What NOT to do here

- Don't edit code in `app/`, `pi-extension/`, `relay/`, `site/`, `cockpit/`
- Don't run subproject build/test commands from here
- To implement something, dispatch via `cmux send` to the target
  subproject's pane (see the [Panes of this cmux workspace](#panes-of-this-cmux-workspace)
  section below). Only ask the user to open a new terminal if the pane is gone.

## Structure

See [README.md](./README.md) for the overview and [plan/](./plan/) for the plans.

## Decisions already taken

Before proposing a change of direction (architecture, pairing, scope, UI, security),
read [`plan/00-decisions.md`](./plan/00-decisions.md). That file lists decisions
closed during exploratory conversations and **they must not be revisited without strong evidence**.

If you still want to revisit one, open an explicit discussion — don't change it silently.

## Plan conventions

- Sequential numbering: `01-bootstrap.md`, `02-ai-orchestration.md`, ...
- Each plan has: Context, Expected structure, Steps with acceptance criteria, DoD, Next plans
- Plans describe **what** + **how to verify**, not the full code
- Pseudocode or exact commands are welcome; the real implementation lives in the subproject

## When to promote a plan to implementation

When the plan has user approval and the steps are concrete enough
for an agent to execute, open Claude in the target subproject and pass the plan as
context. That subproject's agent will follow its own persona.

## Available scouts

To snapshot the state of any subproject before planning, invoke the
Scout subagents in parallel via `Task` — they are read-only and report in a
fixed format:

- `scout-app` — Flutter (`app/`)
- `scout-pi-extension` — Node/TS (`pi-extension/`)
- `scout-relay` — Rust (`relay/`)
- `scout-site` — NextJS (`site/`)
- `scout-cockpit` — Flutter Desktop (`cockpit/`)

Fire several in a single message so they run in parallel. Each report
comes back with Stack & versions, Dependencies, Structure, Health (lint/build/tests)
and detected Smells.

## Panes of this cmux workspace

This workspace ("Remote PI") has 5 dedicated panes — one per subproject — plus this
Orchestrator. Each pane already has a `claude` running in its own session. **Use the
existing panes instead of asking the user to open a new terminal.**

| Pane (title) | Subproject (cwd) |
|---|---|
| `App` | `app/` |
| `Relay` | `relay/` |
| `Extension` | `pi-extension/` |
| `Site` | `site/` |
| `Cockpit` | `cockpit/` |
| `Orchestrator` (you) | monorepo root |

> **Cockpit is the newest pane** and for now it is **started manually** by the
> user — it is **not yet** in `cmux-bootstrap-agents.sh` (which creates the 4
> original ones: App/Relay/Extension/Site). When orchestrating, dispatch to `Cockpit`
> normally when the pane exists; if it's missing, ask the user to start it (don't
> assume the bootstrap script creates it).

> **Never hardcode surface IDs in this documentation.** They change on every
> pane bootstrap. Always resolve them by title via `cmux tree`.

### ⚠️ cmux → Cockpit migration (in progress)

Communication/orchestration is **migrating from cmux to Cockpit** (the 5th
subproject, which now has its own internal CLI — see memory
`project_cockpit_internal_cli`). The new path is preferred; cmux below
stays as a fallback until the migration is done.

**Dispatch via Cockpit** — same protocol (`[ORCH:<id>]`, result file,
`--wait`), only the transport changes:

```bash
# manual pane label (exact match) OR direct tab-id
scripts/cockpit-dispatch.sh Extension 03-ts-codec "Implement step 3 of plan/03-protocol.md"
scripts/cockpit-dispatch.sh --wait Extension 25-wave-x "..."   # run in background
scripts/cockpit-dispatch.sh t319 quick-check "run the tests"  # literal tab-id
```

Key differences vs cmux:
- **Pane resolution by manual LABEL**: give the pane a stable name —
  **double-click the tab** in the terminal (or right-click → "Rename"). That label
  is app-managed and **immune to claude's OSC-title overwrites** (the `✳ <summary>`
  keeps changing only the hidden dynamic `title`). The wrapper resolves name→pane via
  the `label` field of `cockpit list-panes --json` (exact match, case-insensitive).
  Labels must be **unique**; collision = error (never guess a pane — that has already
  caused a dispatch to the wrong pane). "Reset to automatic" in the menu unlocks it.
- **Do NOT resolve by**: `title` (dynamic, claude rewrites it), `workspaceId`
  (it's the workspace root, identical for all in a monorepo), or cwd (volatile, the
  user runs `cd`). Only the `label`. If the pane has no label, pass the tab-id.
- **Tab-ids change on every app boot** — never hardcode them; always `list-panes`.
  (Labels, in contrast, **persist across boots** — anchored to the layout session
  id, not the tab-id.)
- **`--wait` has an extra signal**: in addition to the result-file poll (the
  `INSTRUCTIONS.md` contract), it watches the native `working: true→false` field of
  `list-panes --json` as a backstop (covers an agent that forgot the result file).
- **Solo conversation**: `cockpit send --tab-id <id> "<text>"` +
  `cockpit send-key --tab-id <id> Enter` (Enter separately, same reason as cmux).

Reference skill: [`cockpit-cli`](file:///Users/jacob/.claude/skills/cockpit-cli/SKILL.md).

**Conclusion push (implemented 2026-07-19)**: when the dispatch runs from
inside a Cockpit pane, the script embeds `[ORCH-REPLY:$COCKPIT_PANE_ID]`
in the prompt and the worker (per `INSTRUCTIONS.md`) sends the conclusion directly to
the orchestrator pane via `cockpit send --tab-id <orch>` + Enter — `--wait`
with polling stays as a fallback for a worker that forgets the push.

### Finding the surface ID by title

```bash
# helper: prints the surface:N of the pane titled <Name>
surface_of() {
  cmux tree | awk -v t="$1" '
    $0 ~ "\""t"\"" {
      for (i = 1; i <= NF; i++) if ($i ~ /^surface:/) { print $i; exit }
    }
  '
}

surface_of Extension   # prints the current surface:N
```

### Dispatching a task to a pane (orchestrated mode)

**Always** use the wrapper `scripts/cmux-dispatch.sh`. It resolves the surface
by title, injects `[ORCH:<task-id>]`, and sends + Enter in a single call:

```bash
scripts/cmux-dispatch.sh Extension 03-ts-codec "Implement step 3 of plan/03-protocol.md"
```

Why the wrapper exists: the `[ORCH:<task-id>]` trigger is what makes each
agent enter orchestrated mode (read `.orchestration/INSTRUCTIONS.md`,
respect cwd-only, don't commit). Without the marker, the agent responds in solo
mode. Sending `cmux send` directly to an agent pane is easy to get wrong
(forgot the marker in previous conversations and the user called it out). **Use the wrapper.**

When NOT to use the wrapper:
- Direct exploratory conversation ("what's your role?", "what do you see in X?")
- Debugging, shell commands, resuming claude — solo mode is appropriate
- In those cases use `cmux send --surface "$(surface_of <Name>)" -- "<text>"` +
  `cmux send-key --surface "$(surface_of <Name>)" enter` (Enter separately
  because `\n` becomes a multi-line newline in claude's prompt, not a submit)

### Waiting for the worker to finish (result file polling)

**Preferred form** — dispatch with `--wait` polls
`.orchestration/results/<task-id>.md` until it detects a new mtime. **Always run the
dispatch in the background** (Bash tool with `run_in_background: true`), so the
command shows up in the Claude Code footer and **doesn't block the conversation** — you get
notified when the result file is written and can keep conversating in the meantime:

```bash
# run via Bash with run_in_background: true
scripts/cmux-dispatch.sh --wait Extension 25-wave-x "..."
# doesn't block the turn: runs detached, footer shows progress,
# completion notification arrives when the agent writes the result file
```

> **Why background, never foreground**: `--wait` in the foreground holds the
> entire turn (up to the timeout, default 1800s) and the conversation becomes a hostage of the
> worker. With `run_in_background: true` the polling runs detached — you fire
> N tasks in parallel, they all show up in the footer, and each completion
> notification brings you back to read the result. Foreground only if it's a single,
> quick task you deliberately want to block on (rare).

How it works: the script captures the file's `stat -c %Y` mtime BEFORE the
dispatch (0 if it doesn't exist) and polls every 2s until `cur_mtime > before_mtime`
+ confirms there's a `**Status**:` line. Hook-independent — works
with plain claude in the panes. Default timeout 1800s, tunable via
`--timeout <s>` and `--poll-interval <s>`.

**Why polling instead of hooks**: hooks (`agent.hook.Stop`) are only
emitted when the pane runs `cmux claude-teams`, but our convention is
panes with plain `claude` (per-folder, with its own `.claude/settings.json`
in each subproject). The polling reuses the already-existing result
file convention — the agent is required to write `.orchestration/results/<id>.md` at the
end of any orchestrated task (per `INSTRUCTIONS.md`), so the file
is our de facto "Stop".

**Re-dispatch works**: if a task-id is reused (result file
overwritten), the `before_mtime` snapshot guarantees the next write
still triggers — it's not vulnerable to a pre-existing file.

**Manual form** (debugging, or if you want to watch the file appear):

```bash
# in one terminal: fire without wait
scripts/cmux-dispatch.sh Extension 25-wave-x "..."

# in another: poll yourself
while [ ! -f .orchestration/results/25-wave-x.md ]; do sleep 2; done
cat .orchestration/results/25-wave-x.md
```

**Hooks still work if the pane uses `cmux claude-teams`** — nothing was
removed from cmux, only what our script expects was changed. If the setup ever
becomes claude-teams, `cmux events --category agent --name agent.hook.Stop` remains
valid for anyone who wants to use it.

### Creating the 4 panes from scratch

If the workspace doesn't have the panes yet (or they were closed), use the script
`scripts/cmux-bootstrap-agents.sh`. It creates 4 panes to the right of the current pane,
stacked vertically (App → Relay → Extension → Site), renames each
surface, and dispatches `cd <subproject> && claude [--resume]`.

**You (the orchestrator) should offer to run the script when you notice the panes
are missing.** The user decides whether they want a fresh session or a resumed one — don't
guess for them. Suggested wording:

> "The agent panes aren't in the workspace. Want me to run
> `scripts/cmux-bootstrap-agents.sh`? With `--resume` it resumes the last session of
> each subproject; without the flag it opens claude from scratch."

Ask and wait for the answer before running the script — **never run it yourself
without explicit authorization**, it creates real panes in the user's workspace.

```bash
scripts/cmux-bootstrap-agents.sh           # fresh claude session in each pane
scripts/cmux-bootstrap-agents.sh --resume  # claude --resume (picker)
```

Idempotency: if the 4 panes already exist (by title), the script exits 0 without doing
anything. Mixed state (some exist, others don't) → aborts with an error so you
can clean up manually.

### Closing the 4 panes at once

When the user wants to close all 4 agents (e.g. to recreate from scratch,
or to clean up the workspace), there's a companion script:

```bash
scripts/cmux-close-agents.sh
```

It locates surfaces by title (App / Relay / Extension / Site) in the
current workspace and calls `cmux close-surface` on each. Idempotent: missing
names produce a warning, not an error. Surfaces with other names (Orchestrator, View,
worktrees `✳ <task>...`) are not touched.

**Same rule as bootstrap**: you (the orchestrator) *offer* to run it, never run it
without the user's explicit authorization — the script closes real panes and kills
in-flight claude sessions.

### Clearing context in the 4 panes (starting a new feature)

To start a new feature without dragging along the context of the previous one, **don't recreate
the panes** — just send `/clear` to each agent. `/clear` zeroes out the claude conversation but keeps the process alive in the same folder, same model, same `.claude/`.
Lighter than close+bootstrap; the opposite of `claude --resume` (which would load the
old context).

```bash
scripts/cmux-clear-agents.sh                  # clear all 4
scripts/cmux-clear-agents.sh Extension Site   # clear only those
```

`/clear` is a **solo** command (claude built-in), not an orchestrated dispatch —
that's why the script does **not** use the `[ORCH:]` marker; it sends the literal text +
Enter separately, just like the solo path. Idempotent: missing
titles produce a warning, not an error.

> **Only run this when the agents are idle.** If an agent is in the middle of a task, the
> `/clear` lands as text in the buffer or interrupts the work — wait for the result file
> (or use `--wait` on the dispatch) before clearing. Same courtesy rule as
> bootstrap/close: **offer**, don't run it without the user's ok.

### Reviving a session that died without recreating the pane

If the pane exists but only the `claude` process died, send the command directly:

```bash
sid=$(surface_of App)   # use the helper above
cmux send     --surface "$sid" "cd ~/Projects/remote_pi/app && claude --resume"
cmux send-key --surface "$sid" enter
```

`claude --resume` shows the picker of previous sessions in that folder;
pick the most recent one. Use `claude -c` if you want to skip the picker and jump
straight to the last session. **Always confirm the cwd first** — opening Claude in the
wrong folder breaks the subproject's persona.

### Don't confuse with worktrees

From time to time extra panes named `✳ <task>...` appear — those are worktrees or
temporary sessions generated by other orchestrations (e.g. `/ultrareview`,
background agents). Don't dispatch plan work to them; only the 4 panes
named above are canonical for the planning flow.

## Reporting progress in cmux

cmux accepts visual progress in the workspace via:

- `cmux set-progress <0.0-1.0> --label <text>` — progress bar
- `cmux clear-progress` — clears it
- `cmux set-status <key> <value> [--icon <name>] [--color <#hex>]` — named status

Since we have explicit planning in `plan/`, derive the progress from the
**Definition of Done** checkboxes in each plan:

```bash
# run from the monorepo root
done=$(grep -h "^- \[x\]" plan/*.md | wc -l | tr -d ' ')
total=$(grep -hE "^- \[(x| )\]" plan/*.md | wc -l | tr -d ' ')
pct=$(LC_NUMERIC=C awk "BEGIN { printf \"%.3f\", $done / $total }")  # LC_NUMERIC=C avoids the decimal comma in BR locales
cmux set-progress "$pct" --label "Remote Pi · $done/$total tasks"
```

**When to update**:
- After checking an `[x]` in a DoD
- After adding a new plan (total grows, %% naturally drops)
- After finishing a whole plan: `cmux set-status plan "0N completed" --color "#22c55e"`

**When to clear**:
- When all MVP plans are closed: `cmux clear-progress`

Don't call `set-progress` every turn — only when the real state changed.

## `claude-cmux` skill

For anything beyond basic `set-progress` — dispatch between panes, listening for
`agent.hook.Stop`, notifications, the `.orchestration/` pattern — use the skill
[`claude-cmux`](file:///Users/jacob/.claude/skills/claude-cmux/SKILL.md).

It covers:
- CLI essentials (`send`, `send-key`, `events`, `notify`, `tree`, `list-panes`)
- Automatic variables (`$CMUX_WORKSPACE_ID`, `$CMUX_SURFACE_ID`)
- The orchestration pattern with `INSTRUCTIONS.md` / `plan.md` / `tasks/` / `results/`
- How to use `claude-teams` to emit structured hooks

The skill triggers automatically on cmux questions or requests for parallel
orchestration. Don't duplicate its content here — invoke the skill.
