# Connecting agents (MCP)

Plastic includes an [MCP](https://modelcontextprotocol.io) server so local coding agents —
Claude, Codex, Cursor, GitHub Copilot, VS Code, OpenCode, Antigravity, Gemini CLI, Grok, Windsurf, Cline and others — can read and
write designs and tokens.

Agents work on the same files the editor saves (`workspace/<file>/*.html`, `styles.css`,
`tokens.css`). Every tool call reads the project, applies Plastic's own document operations,
and writes the files back, so agent edits are exactly what the editor would produce. While the
editor is open it picks the change up immediately and shows it live, as one undoable step.
Reading is exact: tools return the saved HTML and CSS, which *is* the design.

## Connect

In the editor, deselect everything and press **Connect more agents** (inspector → MCP). The
dialog has one-click setup for Claude Code and Codex (through their own CLIs) and for Cursor
and VS Code / Copilot (through their install links), plus copy-paste config for every agent.

### macOS desktop

The desktop app starts its own workspace service and Streamable HTTP MCP endpoint at
`http://127.0.0.1:19427/mcp`. Keep Plastic running when using HTTP connections.
**Connect Your Agent** generates configuration for the running app, including its exact
workspace and executable path. Claude Code, Codex and Grok Build offer one-click CLI
registration (`claude mcp add`, `codex mcp add`, `grok mcp add`); Cursor and
VS Code / Copilot offer install links. Gemini CLI, Windsurf, OpenCode, Cline, Antigravity
and Claude Desktop have copyable configuration. Merge the Plastic entry into existing
configuration rather than replacing other servers.

Clients requiring stdio can launch the packaged executable directly:

```bash
"/Applications/Plastic.app/Contents/MacOS/Plastic" --mcp-stdio --workspace "/absolute/path/to/workspace"
```

This does not require Bun or Node. Use the actual installed app and workspace paths shown
in the connection dialog. Each client gets an independent MCP session; all sessions
share the same saved workspace and trigger file-tab AI activity and presence indicators.

The CLI status checks the configured transport and endpoint (or exact command and
workspace arguments); it does not imply a successful live handshake. Restart or refresh
the agent after setup and use its MCP tool list or a Plastic prompt to verify connection.
An existing Claude registration for another endpoint must be removed in its original
scope before adding the desktop connection.

### Development server

Manually, the server is a local command:

```bash
bun /path/to/plastic/server/mcp/stdio.ts
```

For example, with Claude Code:

```bash
claude mcp add --scope user plastic -- bun /path/to/plastic/server/mcp/stdio.ts
```

Use absolute paths; GUI apps often don't have `~/.bun/bin` on their `PATH`. Set
`PLASTIC_WORKSPACE` to point the server at another workspace folder.

While the dev server runs, agents that take a URL can use Streamable HTTP at
`http://localhost:5174/mcp`. It only accepts local requests (the `Host` and any browser
`Origin` must be localhost), which also blocks DNS-rebinding attacks from web pages.

## Tools

| Tool | What it does |
| --- | --- |
| `list_files` | Files in the workspace, with pages and token counts |
| `get_file` | Pages, artboards (id, name, position), class rules, every other rule by selector (`otherRules`) and tokens of one file |
| `get_page` | A page as saved: an outline (tag.classes, `data-pl-id`, text), its HTML, artboard positions, `styles.css` (embedded font data left out) and `tokens.css` |
| `get_node` | One element: tag, attributes, classes with their declarations, every rule that matches it, the winning value of each property and overridden declarations, text, children, HTML |
| `get_screenshot` | A PNG of an artboard or any element in it, rendered exactly as the canvas renders it (same CSS, tokens, fonts and assets; viewport fitted to the artboard). Plastic for Mac renders in a hidden window; the dev server and stdio server use Playwright's Chromium (`PLASTIC_CHROMIUM_PATH` or an installed Chrome also work) |
| `get_frame` | Read a copied frame link: complete subtree HTML, ordered CSS, tokens, node metadata and asset paths |
| `create_file` | A new file (optionally with an empty artboard) |
| `add_frame` | An empty artboard on a page |
| `write_html` | Insert HTML into an element, or onto a page as artboards; `css` merges into `styles.css` rule by rule (an existing selector is updated, not duplicated) |
| `set_page_html` | Replace a page's content; rules for classes the new HTML still uses are kept |
| `update_styles` | Set or remove declarations on a class, on any rule by selector (`.table th`, `.btn:hover`), or on elements' own classes; `remove: true` deletes a rule |
| `set_attributes` / `set_text` | Change attributes (`class` included), the tag or the text of an element |
| `move_node` / `delete_nodes` | Restructure |
| `create_page` / `rename_page` / `delete_page` | Pages (one HTML file each) |
| `get_tokens` / `set_tokens` / `rename_token` | Design tokens; renaming updates every `var()`, deleting replaces each use with the value |
| `import_figma` | Convert a local `.fig` file into a new Plastic file |
| `add_font` | Install a font the codebase uses: its own font files under the family name it declares, or a hosted stylesheet (Google Fonts, Bunny Fonts, Fontshare, CDNFonts) downloaded into the design; optionally sets a font token |
| `add_asset` | Copy a local font (woff2, woff, ttf, otf), SVG, image or video into a file's `assets/`; returns the `assets/<name>` path and how to use it |

The server's instructions tell agents how Plastic files are organized: semantic HTML, one class
per element styled in `styles.css`, tokens as `var(--color-…)`, flex/grid inside artboards.
Class rules are what the inspector edits; descendant, state, pseudo-element and `@media` rules
are kept and render as written. Results of `write_html` and `set_page_html` include `cssMerged`
(existing rules the CSS updated, with the declarations it didn't restate) and warn about classes
no CSS styles.

## Designs recreated from a codebase

When an agent rebuilds an existing screen or component in Plastic, the server's instructions ask
for an exact copy:

- **Structure:** the HTML the app renders, tag for tag: tables as `<table>`/`<thead>`/`<tbody>`/
  `<tr>`/`<th>`/`<td>`, buttons as `<button>`, fields as `<input>`/`<select>`/`<textarea>`, lists,
  headings and landmarks as in the app, with the same attributes and text, and the app's real CSS
  (Tailwind utilities resolved to their values, states and responsive rules included).
- **Fonts:** installed into the design with `add_font` under the exact family name the code
  declares (e.g. "Inter", not next/font's generated name). The codebase's own font files are
  copied in; fonts it loads from Google Fonts or a similar service are downloaded, so the design
  never depends on a font CDN. A hosted `@import` in `write_html` CSS is downloaded automatically.
- **Icons:** each icon's own SVG from the codebase (icon library components in `node_modules`,
  the project's `.svg` files or icon components), inline as `<svg>` with its `viewBox`, paths and
  `currentColor` strokes. Never emoji, Unicode symbols, icon fonts or another icon set.
- **Images and logos:** the real files, copied with `add_asset`.

Agents check their result with `get_screenshot` and compare it with the product before calling a
design done; screenshots of single elements show details at 2x.

`write_html`, `set_page_html` and `update_styles` return `warnings` when a write uses generic
`<div>`/`<span>` elements in place of native ones (roles such as `role="button"`, click handlers,
table-like classes or `display: table-*`), emoji or icon-font markup in place of icons, or a font
the design doesn't have installed, so the agent can correct it.

## Try

- "Create a sign-up page in Plastic with a real form and semantic HTML"
- "Add color and spacing tokens in Plastic and use them in my design"
- "Read my Plastic design and describe its layout and components"
