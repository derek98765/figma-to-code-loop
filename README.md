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

### Optional: project config

The skill will pick up project-specific conventions (dev server URL, design
tokens location, spacing scale, component directory, routing conventions,
etc.) from `.claude/figma-workflow.config.md` in your project root, if
present. See [`figma-workflow.config.example.md`](figma-workflow.config.example.md)
for the format — copy it into your project as `.claude/figma-workflow.config.md`
and fill in your own values. If no config file is found, the skill falls
back to sensible defaults (see `skills/figma-to-code-loop/SKILL.md`).

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
