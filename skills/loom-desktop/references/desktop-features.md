# Desktop Features Reference

Desktop-specific features for Loom apps: file handling, native UX, and access configuration. Read this when adding native platform features to your app.

## Table of Contents

- [File Drag-and-Drop](#file-drag-and-drop)
- [Native File Dialogs](#native-file-dialogs)
- [System Tray](#system-tray)
- [Native Menus](#native-menus)
- [Local File Access Configuration](#local-file-access-configuration)

---

## File Drag-and-Drop

Drag-and-drop supports **text files only**. PDFs, images, and other binary
files cannot be read via `FileReader.readAsText()` — doing so produces mojibake
injected directly into the prompt. For PDFs and images, direct the user to
the native **Open File…** dialog (see [Native File Dialogs](#native-file-dialogs)),
which returns a real filesystem path that Claude's `Read` tool handles natively
(PDFs and images included).

**Why text-only?** `File.path` is an Electron-specific extension — it does NOT
exist in WKWebView (macOS), WebView2 (Windows), or WebKit2GTK (Linux). System
webviews can't expose a dropped file's real path (gotcha #12 in `references/gotchas.md`),
so the only in-webview option is `FileReader.readAsText()`, which only works
for text. For binary files the correct path is: native dialog → filesystem path
→ Claude's `Read` tool.

**Webview-side: Text-type guard**

```typescript
const TEXT_EXT = /\.(txt|md|markdown|json|ya?ml|csv|tsv|js|ts|tsx|jsx|py|rb|go|rs|java|c|h|cpp|sh|html|css|xml|log)$/i;

function isTextFile(file: File): boolean {
  return file.type.startsWith("text/") || file.type === "application/json" || TEXT_EXT.test(file.name);
}
```

**Webview-side: Drop handler (text-only, binary redirected to Open File…)**

```tsx
type DroppedFile = { name: string; content: string };

function DropZone({
  onFilesRead,
  onStatusMessage,
}: {
  onFilesRead: (files: DroppedFile[]) => void;
  onStatusMessage: (msg: string) => void;
}) {
  const [dragging, setDragging] = useState(false);

  async function handleDrop(fileList: FileList) {
    const textFiles: DroppedFile[] = [];
    for (const file of Array.from(fileList)) {
      if (isTextFile(file)) {
        const content = await new Promise<string>((resolve, reject) => {
          const reader = new FileReader();
          reader.onload = () => resolve(reader.result as string);
          reader.onerror = () => reject(reader.error);
          reader.readAsText(file);
        });
        textFiles.push({ name: file.name, content });
      } else {
        // System webviews can't expose a dropped file's path (gotcha #12), and reading a
        // PDF/image as text yields garbage. Send the user to the native Open File dialog,
        // which returns a real path that Claude reads natively.
        onStatusMessage(
          `${file.name}: add PDFs/images via "Open File…" — drag-and-drop only supports text files.`
        );
      }
    }
    if (textFiles.length > 0) onFilesRead(textFiles);
  }

  return (
    <div
      className={`drop-zone ${dragging ? "active" : ""}`}
      onDragOver={(e) => { e.preventDefault(); setDragging(true); }}
      onDragLeave={() => setDragging(false)}
      onDrop={async (e) => {
        e.preventDefault();
        setDragging(false);
        await handleDrop(e.dataTransfer.files);
      }}
    >
      Drop text files here to analyze (PDFs/images: use Open File…)
    </div>
  );
}
```

**Bun-side: Incorporating file content into prompts**

The webview sends text file content (not paths) via RPC. Include it directly
in the prompt — Claude doesn't need filesystem tools to read it:

```typescript
// In the startTask handler — files are { name, content } objects:
const fileContext = files.length > 0
  ? files.map((f) => `\n\n--- ${f.name} ---\n${f.content}`).join("")
  : "";
const fullPrompt = prompt + fileContext;
```

Since file content is embedded in the prompt, Claude doesn't need `Read`
tool access for dropped text files. For PDFs and images, use the Open File…
dialog — it returns a filesystem path, and Claude's `Read` tool handles those
natively without the `dontAsk` limitation.

---

## Native File Dialogs

Open files for input, save results to disk.

**Bun-side: Open file dialog**

```typescript
import { Utils } from "electrobun/bun";

// Add to RPC schema bun.requests:
// openFile: { params: {}; response: { path: string | null } };

// Handler:
openFile: async () => {
  const result = await Utils.openFileDialog({
    title: "Select a file to analyze",
    filters: [
      { name: "All Files", extensions: ["*"] },
      { name: "Code", extensions: ["ts", "js", "py", "rs", "go"] },
      { name: "Documents", extensions: ["md", "txt", "pdf"] },
    ],
  });
  return { path: result ?? null };
},
```

**Saving results to disk**

ElectroBun doesn't have `saveFileDialog()` yet. Use `Bun.write()` with
a known path, or ask the user for a filename via RPC:

```typescript
// Add to RPC schema bun.requests:
// saveResult: { params: { content: string; filename: string }; response: { saved: boolean } };

// Handler:
saveResult: async ({ content, filename }) => {
  const desktopPath = `${process.env.HOME}/Desktop/${filename}`;
  await Bun.write(desktopPath, content);
  return { saved: true };
},
```

For a better UX, use `Utils.showItemInFolder()` after saving to open Finder
with the file highlighted:

```typescript
await Bun.write(desktopPath, content);
Utils.showItemInFolder(desktopPath);
```

---

## System Tray

The full system tray implementation is in `desktop-patterns.md#pattern-4-background`.
Key points:

- `new Tray({ title, image, width, height })` — create tray icon
- `tray.setMenu(items)` — update menu items dynamically
- `tray.on("tray-clicked", handler)` — handle tray icon click
- `tray.on("tray-item-clicked", handler)` — handle menu item click
- Use `views://` scheme for icon paths to reference bundled assets

---

## Native Menus

Define the app menu bar with keyboard shortcuts:

```typescript
import { ApplicationMenu } from "electrobun/bun";

ApplicationMenu.setApplicationMenu([
  {
    label: "File",
    submenu: [
      { label: "New Task", action: "new-task", accelerator: "n" },
      { label: "Open File...", action: "open-file", accelerator: "o" },
      { type: "separator" },
      { label: "Quit", role: "quit" },
    ],
  },
  {
    label: "Edit",
    submenu: [
      { label: "Undo", role: "undo" },
      { label: "Redo", role: "redo" },
      { type: "separator" },
      { label: "Cut", role: "cut" },
      { label: "Copy", role: "copy" },
      { label: "Paste", role: "paste" },
      { label: "Select All", role: "selectAll" },
    ],
  },
  {
    label: "Task",
    submenu: [
      { label: "Abort Current", action: "abort", accelerator: "." },
      { type: "separator" },
      { label: "Model: Haiku", action: "model-haiku" },
      { label: "Model: Sonnet", action: "model-sonnet", checked: true },
      { label: "Model: Opus", action: "model-opus" },
    ],
  },
]);

// Handle menu actions
ApplicationMenu.on("application-menu-clicked", (e) => {
  switch (e.data.action) {
    case "new-task":
      // Focus the input field via RPC or reload the view
      break;
    case "open-file":
      // Trigger file dialog
      break;
    case "abort":
      // Abort current task
      break;
    case "model-haiku":
    case "model-sonnet":
    case "model-opus":
      // Update model preference
      break;
  }
});
```

The `accelerator` property maps to Cmd+key on macOS and Ctrl+key on Windows.
Single-character accelerators are supported on all platforms. The `role` property
provides built-in OS clipboard and window management.

Note: Application menu is fully supported on macOS. Windows supports basic
accelerators. Linux does not currently support application menus.

---

## Local File Access Configuration

Claude's tool access determines what it can do on the user's machine. Match
the permission set to the app's purpose:

| Use Case | `--tools` Value | Notes |
|---|---|---|
| Read-only analysis | `"Read,Glob,Grep"` | Safe default for file inspection |
| Code modification | `"Read,Edit,Write,Glob,Grep,Bash"` | Full dev toolkit |
| Pure reasoning | Omit `--tools` entirely | Don't use `--tools ""` — it's fragile |
| Web research | `"WebSearch,WebFetch"` | Internet access, no local files |
| Scoped read | `--allowedTools "Read(/src/**)"` | Only specific directories |

Use `--permission-mode bypassPermissions` for desktop apps. `dontAsk`
auto-denies anything not whitelisted AND blocks reads outside the project
directory — dropped files from the user's desktop or home folder won't be
accessible. Since desktop Loom apps control the prompt and tools list,
`bypassPermissions` with an explicit `--tools` list is the right choice.

For **text files** the user drops, the better approach is to read them in the
webview (via `FileReader.readAsText()` → RPC) and include the content directly
in the prompt. For **PDFs and images**, use the native Open File… dialog
(`Utils.openFileDialog()`) to get a real filesystem path, then let Claude's
`Read` tool handle it with `bypassPermissions`. See the
[File Drag-and-Drop](#file-drag-and-drop) and
[Native File Dialogs](#native-file-dialogs) sections.
