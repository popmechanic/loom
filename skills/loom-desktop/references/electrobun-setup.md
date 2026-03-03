# ElectroBun Project Setup

A focused guide to scaffolding an ElectroBun project for Loom desktop apps.

## Prerequisites

- **Bun** v1.1+ — [bun.sh](https://bun.sh)
- **Claude CLI** installed and on PATH — `claude --version` should return a version string
- **Git** — for version control and submodule support
- **Platform**: macOS 14+, Windows 11+, or Ubuntu 22.04+ (other Linux with gtk3 & webkit2gtk-4.1)

## Creating a New Project

ElectroBun provides an interactive initializer with starter templates:

```bash
bunx electrobun init
```

This prompts you to choose a template. Available templates:

| Template | Description |
|----------|-------------|
| `hello-world` | Minimal app — start here for Loom desktop apps |
| `photo-booth` | Camera integration example |
| `interactive-playground` | API exploration sandbox |
| `multitab-browser` | Multi-tab browser with navigation |

To skip the prompt and choose directly:

```bash
bunx electrobun init hello-world
```

After initialization, install dependencies:

```bash
cd myapp
bun install
```

## Project Structure

An ElectroBun project has this layout:

```
myapp/
├── electrobun.config.ts      # App config: name, version, build options
├── package.json
├── src/
│   ├── bun/                  # Main process (runs in Bun)
│   │   └── index.ts          # Entry point — spawns windows, handles RPC
│   └── mainview/             # Webview code (runs in browser context)
│       └── index.ts          # Entry point — renders UI, sends RPC requests
└── tsconfig.json
```

**Two processes, one language:**
- `src/bun/` runs in the Bun runtime. It has full system access — filesystem, child processes, native APIs. This is where you spawn `claude -p`.
- `src/mainview/` runs in a system webview (WKWebView on macOS, WebView2 on Windows, WebKit2GTK on Linux). It's a browser context with DOM access. This is where you render the UI.

They communicate via typed RPC over an encrypted WebSocket — no manual HTTP layer needed.

## Configuration

The `electrobun.config.ts` file defines your app:

```typescript
import type { ElectrobunConfig } from "electrobun/config";

export default {
  app: {
    name: "my-claude-app",
    identifier: "com.example.my-claude-app",
    version: "1.0.0",
  },
  runtime: {
    exitOnLastWindowClosed: true,
  },
  build: {
    bun: {
      entrypoint: "src/bun/index.ts",
    },
    views: {
      mainview: {
        entrypoint: "src/mainview/index.ts",
      },
    },
  },
} satisfies ElectrobunConfig;
```

Key sections:
- **`app`** — Name, bundle identifier, version. The identifier must be unique for code signing.
- **`runtime`** — Behavior settings. `exitOnLastWindowClosed` controls whether the app quits when all windows close.
- **`build.bun`** — Main process entry point. Accepts all `Bun.build()` options (plugins, external, sourcemap, etc.).
- **`build.views`** — Named webview entry points. Each view is bundled separately.

## Development Workflow

### Start in dev mode

```bash
bunx electrobun dev
```

This builds and launches the app. Add `--watch` for auto-rebuild on source changes:

```bash
bunx electrobun dev --watch
```

File changes trigger a rebuild after a 300ms debounce.

### Run without rebuilding

```bash
bunx electrobun run
```

Launches the last-built dev bundle. Useful for quick relaunches without recompilation.

### Build for distribution

```bash
bunx electrobun build --env=stable
```

Build environments:
| Environment | Purpose | Code signing |
|-------------|---------|--------------|
| `dev` (default) | Fast iteration, terminal output | None |
| `canary` | Pre-release builds | Optional |
| `stable` | Production distribution | Full |

## Key Concepts

### BrowserWindow

Create native windows from the Bun process:

```typescript
import { BrowserWindow } from "electrobun/bun";

const win = new BrowserWindow({
  title: "My Claude App",
  frame: { width: 1200, height: 800, x: 100, y: 100 },
  url: "views://mainview/index.html",
  rpc: myRPC,
});
```

The `views://` URL scheme loads bundled webview content. The `rpc` option connects the typed RPC channel.

### BrowserView

Embed web views within windows. Supports the same options as BrowserWindow (url, html, rpc, preload, partition).

```typescript
import { BrowserView } from "electrobun/bun";

const rpc = BrowserView.defineRPC<MySchema>({
  handlers: {
    requests: {
      startTask: async ({ prompt }) => { /* ... */ },
    },
  },
});
```

### RPC (Remote Procedure Calls)

The typed bridge between Bun and webview. Define a schema, register handlers on the Bun side, call them from the webview:

- **Bun side**: `BrowserView.defineRPC<Schema>({ handlers: { requests: { ... } } })` — define request handlers, send messages via `rpc.sendProxy.*`
- **Webview side**: `Electroview.defineRPC<Schema>({ handlers: { messages: { ... } } })` + `new Electroview({ rpc })` — receive messages via handlers, call requests on the Bun process. **Note:** `new Electroview<T>()` (generic-only form) crashes — must pass `{ rpc }`

See `@references/rpc-schema-reference.md` for the full typed contract.

### Preload Scripts

JavaScript that runs in the webview before page scripts. Set via the `preload` option on BrowserWindow or BrowserView.

### Navigation Rules

Control which URLs the webview can navigate to:

```typescript
view.setNavigationRules([
  "https://trusted-domain.com/*",  // Allow
  "^https://blocked.com/*",        // Block (^ prefix)
]);
```

## Next Steps

With the project scaffolded and running:
1. Define your RPC schema — see `@references/rpc-schema-reference.md`
2. Implement the Claude Manager in `src/bun/` — see the patterns in SKILL.md
3. Build the UI in `src/mainview/`

For the complete ElectroBun API, see [blackboard.sh/electrobun/docs](https://blackboard.sh/electrobun/docs/).
