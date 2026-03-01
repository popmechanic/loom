# cli-app

A Claude Code plugin that teaches Claude how to build web applications using `claude -p` as the runtime.

Instead of a traditional API backend, a server spawns Claude processes that read files, run commands, and return structured output. A custom frontend renders results as rich UI — dashboards, charts, timelines, annotated views.

## Installation

```
/plugin marketplace add marcusestes/cli-wrapper
/plugin install cli-app
```

## What it builds

Apps where Claude is the backend:
- Analytics dashboards
- Monitoring tools
- Code review UIs
- Data explorers
- Any app calling `claude -p` instead of a traditional API

## Example prompts

- "Build a web app that uses Claude to analyze my codebase"
- "Create a dashboard that runs Claude commands and shows results"
- "Make a code review UI powered by Claude"

## Development

The following directories are used for development and are not part of the plugin:

- `commit-storyteller/` — demo app
- `cli-app-workspace/` — eval workspace
