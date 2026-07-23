# figma-to-code-loop

1 skill + 2 agents to convert a Figma design to code, with a built-in
implement/review loop.

- **Skill:** `figma-to-code-loop` — reads a Figma file/frame link, groups
  frames into pages, writes a to-do list, then orchestrates the two agents
  below per task until each page passes review (or hits its round cap).
- **Agents:**
  - `figma-implementor` — builds one page (across all its states) from Figma
    design context.
  - `figma-reviewer` — independently checks the built page against Figma in
    the live browser (Claude for Chrome) and reports pass/fail.

## Prerequisites

`figma-reviewer` reviews every page live in a real browser using the
**Claude for Chrome** extension — this is fixed and not project-configurable
(see `agents/figma-reviewer.md`). That means:

- You must use the **Claude Code CLI**, not the Claude desktop app. The
  Claude for Chrome extension is driven through the CLI's browser
  integration, which the desktop app does not expose.
- Before running this workflow, `cd` into your project and start Claude Code
  with the Chrome flag as your first step:

  ```bash
  cd /path/to/your/project
  claude --chrome
  ```

  Skipping `--chrome` means `figma-reviewer` has no way to drive or inspect
  the live page, and the review step will fail.

## Installation

Claude Code loads skills and agents from `~/.claude/skills/` and
`~/.claude/agents/` (global, all projects) or `.claude/skills/` and
`.claude/agents/` (project-local). To install:

1. Copy `skills/figma-to-code-loop/` into your global `~/.claude/skills/`
   folder (or a project's `.claude/skills/`).
2. Copy both files from `agents/` into your global `~/.claude/agents/`
   folder (or a project's `.claude/agents/`).

```bash
cp -r skills/figma-to-code-loop ~/.claude/skills/
cp agents/figma-implementor.md agents/figma-reviewer.md ~/.claude/agents/
```

Restart Claude Code (or start a new session) so it picks up the new skill
and agents.

## Project config

This skill is generic by default — it doesn't know your dev server URL, your
design token file, your component conventions, or your routing setup. Rather
than asking you those questions on every run, it reads them once from an
optional config file: `.claude/figma-workflow.config.md` in your project
root.

**What it's for:** giving `figma-to-code-loop` (and the agents it spawns)
enough project context to build pages that actually fit your codebase —
right dev server to preview against, right tokens instead of hardcoded
values, right place to put new pages, right review-round budget — without
you repeating that context in every prompt.

**How to set it up:** copy
[`figma-workflow.config.example.md`](figma-workflow.config.example.md) into
your project as `.claude/figma-workflow.config.md` and fill in your project's
real values in place of the placeholders.

```bash
mkdir -p .claude
cp figma-workflow.config.example.md /path/to/your/project/.claude/figma-workflow.config.md
```

**What the skill actually reads from it:**

- `dev_server_url` — used directly by the skill and passed to
  `figma-implementor`/`figma-reviewer` so they preview and review against the
  right running app.
- `max_review_rounds` — used directly to cap how many implement/review
  rounds a page gets before it's logged as blocked instead of looped on
  forever.
- `todo_file` / `pending_issues_file` — if set, used as a fixed, shared
  path instead of the default per-flow-slug file naming. Only set these if
  you deliberately want one shared to-do list across multiple Figma flows.
- Everything else in the file (`tokens_file`, `spacing_scale`,
  `ui_components_dir`, page/routing conventions) is contextual guidance
  passed along for the implementor/reviewer agents to follow — not parsed
  into skill logic, but still worth filling in accurately.
- `review_tool` is informational only. `figma-reviewer` always uses the
  Claude for Chrome extension; this can't be changed per project.

**If the file doesn't exist**, the skill falls back to built-in defaults
(per-flow-slug to-do/pending-issues files, 3 review rounds) and asks you for
the dev server URL if it can't find one in `CLAUDE.md`/`AGENTS.md`. See
`skills/figma-to-code-loop/SKILL.md` (Step 0) for the exact fallback
behavior.

## Usage

1. Start Claude Code with Chrome enabled, from your project directory (see
   Prerequisites above):

   ```bash
   cd /path/to/your/project
   claude --chrome
   ```

2. Invoke the skill with a Figma file or frame link:

   ```
   /figma-to-code-loop <figma link>
   ```

It's designed to be re-entered repeatedly via `/loop` — each call does one
unit of work (read the flow, implement a page, or run one review round) and
stops. See `skills/figma-to-code-loop/SKILL.md` for the full workflow.
