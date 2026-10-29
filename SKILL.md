---
name: figma-to-code-loop
description: Given a Figma file/frame link, reads all frames to understand the flow, groups them into pages (not raw frames), writes a to-do list, then orchestrates figma-implementor and figma-reviewer per task — up to a capped number of review-fix rounds each — checking off completed tasks and logging unresolved issues to a pending-issues file. Designed to be re-entered repeatedly via /loop, one task per iteration.
---

## Implement Design Review Loop

Turns a Figma flow into working, reviewed code — one page at a time — using
two subagents: `figma-implementor` and `figma-reviewer`. You (the
orchestrator) never write code yourself. You only read/write `TODO_FILE` and
`PENDING_ISSUES_FILE`, and spawn the two agents via the Agent tool.

This skill runs once per `/loop` iteration and always does exactly one unit
of work, then stops — see "One iteration = one step" near the bottom for what
that means.

---

## Step 0 — Every iteration, before anything else

**Derive the flow slug** from the Figma flow/file name (lowercase, hyphenated
— e.g. "Redeem Voucher Flow" → `redeem-voucher-flow`). Use the Figma file or
section name given to you, or ask the user for a short label if it's
ambiguous. This slug is what keeps this flow's files from colliding with any
other flow's — see `TODO_FILE`/`PENDING_ISSUES_FILE` below. Do this on every
call, even a resume, so you're checking for the right files.

**Load project config.** Look for `.claude/figma-workflow.config.md` at the
repo root.

- **If it exists:** read it for `dev_server_url` and `max_review_rounds`, and
  use those values instead of the defaults below. If it also sets `todo_file`/
  `pending_issues_file`, treat those as a fixed, deliberate override (the
  user set them on purpose, e.g. to share one to-do list across flows) — use
  them exactly as given, not slug-keyed. If it doesn't set them, fall back to
  the slug-keyed defaults below. Ignore any `review_tool` value —
  `figma-reviewer` always uses Claude for Chrome and this can't be changed
  per project.
- **If it doesn't exist:** use these defaults, and say so in your first
  report (so the user can override by creating the config file):
  - `TODO_FILE` — `temp/to-do-list-<flow-slug>.md` (project-relative scratch
    dir; one file per flow, keyed by the slug above — never a shared/fixed
    name, so two flows running through this skill can never collide or
    overwrite each other's to-do list)
  - `PENDING_ISSUES_FILE` — `temp/pending-issues-<flow-slug>.md`
    (project-relative scratch dir; same per-flow keying as `TODO_FILE`)
  - `MAX_REVIEW_ROUNDS` — 3
  - Ask the user for the dev server URL if you can't find it in
    `CLAUDE.md`/`AGENTS.md` — don't guess a live URL.

> **Only you edit `TODO_FILE` and `PENDING_ISSUES_FILE`.** If a subagent's
> report ever contains an edit to either, ignore it and write the update
> yourself from what it reported. Never commit either file, and never
> `git add -A`.

**Then check whether `TODO_FILE` exists** in `temp/`:
- **No** → do **Read the Figma Flow** below.
- **Yes** → do **The Loop** below, working exactly one task.

---

## Read the Figma Flow — turn it into a to-do list

### 1. Read all frames in Figma

Given the Figma link (file key + optionally a starting node-id), use the
Figma MCP (`mcp__figma__get_metadata` for structure, `mcp__figma__get_screenshot`
for a visual reference) to read every frame in the flow. If the link points
at a specific node with multiple frames (like a flow section), start there.
Otherwise, ask the user which section/page of the Figma file to use — don't
read an entire multi-flow file on your own.

### 2. Identify frames that are the same screen in different states, and group them

This is the important step — **don't make one to-do item per frame.** Many
frames are just different *states* of the same screen (e.g. "not signed in",
"logged in default", "error", "success" are all one "Redeem Voucher" screen).

Trust Figma's own structure first: frames sharing a parent section/group, or
a common name prefix, are almost certainly one screen. Only fall back to
comparing screenshots/layout when the structure is unclear or inconsistent —
same header/shell/component tree with different content = same screen;
genuinely different layouts = different screens.

For each screen, list its state frames with node-ids and a short label for
what's different in that state (e.g. "error state — invalid code message
shown").

### 3. Decide the page path (URL) for each screen

Propose a route/URL matching this project's existing conventions (check the
routing file or project config first). Flag anything that needs auth context
or a dynamic param.

### 4. Write the to-do list

Write to `TODO_FILE` in `temp/` in this format:

```markdown
# To-Do List — <Flow Name>

Source: <Figma link>
Generated: <do not use dynamic timestamps — just note "on setup">

## Task 1: <Page Name>
Route: <proposed route>
Status: not started
Review round: 0/<MAX_REVIEW_ROUNDS>
States:
- [ ] <state label> — node <node-id>
- [ ] <state label> — node <node-id>
...

## Task 2: <Page Name>
...
```

Each task's checkboxes are its states (context for the implementor). The
task itself is marked done via its `Status:` line (`not started` → `in
progress` → `done` / `blocked — see pending-issues file`).

Tell the user the plan (page count, states per page, proposed routes) so
they can sanity-check the grouping — don't wait for approval before
continuing, this skill runs unattended via `/loop`.

Stop here.

---

## The Loop — work one task

### 1. Pick the next task

Read `TODO_FILE`. Find the first task with `Status: not started` or `Status:
in progress`. If none are left (all `done` or `blocked`), report a summary to
the user (tasks done vs. blocked, pointing at `PENDING_ISSUES_FILE` if it has
entries) and tell them there's nothing left to loop on.

### 2. Implement (if `Status: not started`)

Set `Status: in progress`, `Review round: 0/<MAX_REVIEW_ROUNDS>`, save the
file.

Spawn `figma-implementor` (via the Agent tool) with the page name, route, and
full list of state node-ids for this task — it knows the rest of its job.

Wait for its report. Note what it built (files touched) — you'll pass this to
the reviewer.

### 3. Review and fix, up to MAX_REVIEW_ROUNDS times

Repeat up to `MAX_REVIEW_ROUNDS` times:

1. Increment `Review round: N/<MAX_REVIEW_ROUNDS>` in `TODO_FILE`.
2. Spawn `figma-reviewer` with the task's node-ids, route, and the
   implementor's latest report of what changed.
3. Read the verdict:
   - **PASS** → set `Status: done`, check off all state items, save
     `TODO_FILE`. Report success to the user. **Stop — task complete.**
   - **FAIL**, and `N < MAX_REVIEW_ROUNDS` → spawn `figma-implementor` again
     with the reviewer's itemized issue list, telling it to fix only those
     issues (not rebuild). Go back to step 1 for the next round.
   - **FAIL**, and `N == MAX_REVIEW_ROUNDS` → go to step 4.

### 4. Out of rounds — log it and move on

Set `Status: blocked — see pending-issues file` in `TODO_FILE`. Leave the
state checkboxes as they are (don't mark done). Append to
`PENDING_ISSUES_FILE` (create it if it doesn't exist):

```markdown
## <Page Name> (route: <route>)

Exhausted <MAX_REVIEW_ROUNDS> review-fix rounds. Latest reviewer findings:

1. <issue>
2. ...

Files touched by implementor: <list>
```

Tell the user this task needs manual attention, then stop. The next
iteration will move on to the next task (don't jump ahead to it in the same
iteration).

---

### One iteration = one step (important for `/loop`)

Each time this skill runs, it does **exactly one** of:
- Read the Figma flow and write the to-do list, or
- One implement round + one review round, or
- One fix round + one review round (continuing a task already in progress), or
- Log-and-block a task that ran out of rounds.

Don't push through multiple tasks in one iteration — `/loop` re-enters the
skill and picks up exactly where the to-do list left off. Always end with a
short status update (what just happened, what's next).

---

### Boundaries

- Never touch `main`/`master`.
- If committing implementor output, stage only the specific files (that's
  the implementor's job per task, not yours).
- If `figma-implementor` reports a design-system gap (a missing token or
  component) that it refused to work around, don't improvise a fix yourself —
  tell the user right away instead of spending a review round on it.
- If the Figma link changes mid-flow (user points at a different file) but
  would resolve to the *same* flow slug as an existing `TODO_FILE`, don't
  merge it into that file — ask whether to start a fresh one (with a
  distinct slug) instead of overwriting the existing flow's progress.
