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

## Usage

Invoke the skill with a Figma file or frame link:

```
/figma-to-code-loop <figma link>
```

It's designed to be re-entered repeatedly via `/loop` — each call does one
unit of work (read the flow, implement a page, or run one review round) and
stops. See `skills/figma-to-code-loop/SKILL.md` for the full workflow.
