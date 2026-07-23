# Figma Workflow Config

Project-specific config for the `figma-to-code-loop` skill and its
`figma-implementor` / `figma-reviewer` agents. Copy this file to
`.claude/figma-workflow.config.md` in your project root and fill in your own
values — everything below is a placeholder.

## Dev server

```
dev_server_url: https://your-dev-server.example.com
```

The local (or preview) URL where the app runs, so `figma-implementor` and
`figma-reviewer` can open live pages.

## Design tokens

```
tokens_file: src/styles/tokens.css
```

Where colors, spacing, typography, and other design tokens are defined.
Point this at your project's actual token source so agents reuse tokens
instead of hardcoding values.

## Spacing scale

```
spacing_scale: 4, 8, 12, 16, 24, 32, 48, 64, 96
```

If your project uses a custom/restricted spacing scale (e.g. a Tailwind
config with only specific step values allowed), list the valid values here.
Delete this section if your project uses a standard/unrestricted scale.

## UI components

```
ui_components_dir: src/components/ui/
```

Directory of existing reusable UI components (e.g. a shadcn/ui setup).
Agents should check here before creating new one-off components.

## Page convention

```
page_path_convention: src/pages/<PageName>/index.tsx
page_examples: src/pages/ExamplePageOne, src/pages/ExamplePageTwo
routing_file: src/App.tsx
```

Where new pages should live, an example of the pattern, and the file where
routes are registered.

## Working-state files (orchestrator-owned, never committed)

```
todo_file: temp/to-do-list.md
pending_issues_file: temp/pending-issues.md
max_review_rounds: 3
```

Leave `todo_file`/`pending_issues_file` unset to let the skill default to
per-flow-slug files (recommended for most projects — see SKILL.md). Only set
them here if you deliberately want one shared to-do list across flows.

## Review method

```
review_tool: chrome-extension
```

Informational only — `figma-reviewer` always uses the Claude for Chrome
browser extension and this value is not currently configurable per project.
