# Architecture

OpenIDEA is one backend with two frontends over the same IPC surface: the desktop
shell (`src-tauri`, Tauri 2) and the remote server (`openidea-server`, JSON-RPC).
This page is the map from a user action to the code that runs it, and the rules
that keep the layers apart. Contributor rules and commands are in
[AGENTS.md](../AGENTS.md); this file explains the shape they assume.

## Three layers

| Layer      | Lives in                          | Does                                                                                                                                                                              |
| ---------- | --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| UI         | `src/` — React, CodeMirror, xterm | Rendering, keymaps, actions, editor decorations. Never touches the disk, a process or a socket directly.                                                                          |
| Shell glue | `src-tauri/`                      | Windows, menus, native dialogs, and one `commands/<domain>.rs` per domain: parse arguments, move the work off the async runtime, forward streaming and events. No domain logic.   |
| Backend    | `crates/ide-*`                    | Every feature and all of its state. `ide-host` assembles the crates into a `Host`; `openidea-server` serves the same `Host` over JSON-RPC so remote development reuses all of it. |

The UI and the backend share no code. `src/bindings.ts` is generated from the
Rust command and type definitions by tauri-specta (`pnpm gen:bindings`) and is
never edited by hand; CI fails when the committed file differs from a fresh
generation.

## One action, end to end

Following "Go to File" (`⌘⇧O` in the IntelliJ keymap) from the keystroke to the
index:

1. `src/platform/keymaps/*.ts` maps the shortcut to an action id; the dispatcher in
   `src/platform/actions.ts` runs it. Every user-invocable behaviour is an action
   with a stable id, so it is also searchable and rebindable. Go to File is
   `navigate.gotoFile`, registered in `src/features/search/index.ts`.
2. The action opens Search Everywhere, whose file provider calls the typed IPC
   surface: `commands.findFiles(query, limit)` from `src/platform/ipc.ts`, typed
   against `src/bindings.ts` (`src/features/search/everywhere/deps.ts` binds it).
3. The shell receives it in `src-tauri/src/commands/nav.rs`, which is a few lines:
   resolve the window's project, `spawn_blocking`, call the backend.
4. `crates/ide-host/src/ops/nav.rs` is the implementation; it asks the project's
   index (`crates/ide-index`, redb + nucleo) and returns plain data.
5. The same command is registered once in `src-tauri/src/commands/mod.rs`, both for
   the desktop IPC surface and for the remote server's dispatcher.

So the layers a new feature usually touches, from the outside in, are:
`src/features/<name>/index.ts` → `src/platform` (actions, ipc) →
`src-tauri/src/commands/<domain>.rs` → `crates/ide-host/src/ops/<domain>.rs` →
`crates/ide-<domain>`.

About 490 commands are wired up this way today.

## Crates

One crate per domain, 35 in total. All of them depend on `ide-core` for shared
types; a crate never reaches into another crate's internals. `ide-core` itself
depends on nothing in the workspace, which is what keeps the graph acyclic.

| Role                  | Crates                                                                                                                                                                                                                                                    |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Shared vocabulary     | `ide-core` (ids, text coordinates, events, errors, settings)                                                                                                                                                                                              |
| Text and files        | `ide-text` (rope documents, change sets), `ide-vfs` (listing, encoding, watching, archives), `ide-workspace` (documents, settings layering, local history, roots)                                                                                         |
| Language and protocol | `ide-language` (language definitions, tree-sitter), `ide-extension` (plugin kernel), `ide-syntax` (structure service when no server answers), `ide-jsonrpc` (framing and peer), `ide-lsp`, `ide-dap`, `ide-tools` (language server and adapter installer) |
| Insight               | `ide-index`, `ide-search`, `ide-analysis`, `ide-spell`                                                                                                                                                                                                    |
| Execution             | `ide-run` (configurations and processes), `ide-coverage`, `ide-profiler`, `ide-build`, `ide-sdk` (interpreters and toolchains), `ide-terminal`                                                                                                            |
| Integrations          | `ide-vcs`, `ide-github`, `ide-ssh`, `ide-remote`, `ide-db`, `ide-http`, `ide-docker`, `ide-agent`, `ide-notebook`, `ide-web`, `ide-clipboard`, `ide-fonts`                                                                                                |
| Extension runtime     | `ide-plugin` (wasmtime)                                                                                                                                                                                                                                   |
| Assembly              | `ide-host` (`ops/`, `rpc`, `plugins`, `trust`, per-window projects)                                                                                                                                                                                       |

The dependency direction the plugin system establishes is written down in
`crates/ide-extension/src/lib.rs`: `ide-core` ← `ide-extension` ← feature crates
and `ide-plugin` ← `ide-host`. Feature crates depend on the kernel for the
contribution types; only the assembly layer knows about the Wasm runtime.

Reads and writes follow one pattern throughout: fast reads use a library in the
process (gitoxide for Git reads, tree-sitter for syntax, ripgrep's engine for
search), while anything that must behave exactly like the command line shells out
to it (the `git` CLI for commits and pushes, `docker`, `gh`).

## Conventions that cross every layer

- **Coordinates.** Lines are 0-based, columns and offsets are UTF-16, everywhere
  (`ide-core::text`). Adapters that speak a 1-based protocol convert at their own
  edge: `ide-lsp` and `ide-dap` do.
- **Errors.** Every command returns `IdeResult<T>`; internal errors are converted
  at the crate edge, so the UI receives `{kind, message}` and can react to `kind`.
- **Streaming.** Long output goes over `tauri::ipc::Channel<T>`.
- **Cancellation.** The UI allocates a `RequestId` and calls `cancel_request`; the
  backend honours a `CancellationToken`.
- **Events.** Backend-to-UI notifications travel as `IdeEvent` through an
  `EventSink`. Events are scoped to the window they concern.
- **Blocking work** never runs on the UI thread or on a Tauri command thread.

## Frontend structure

- `src/features/<name>/` — one feature, exporting `activate()`. Features are
  activated in a fixed order by `src/features/index.ts`; several also export
  `activateSettingsPages()` for the settings window.
- Features never import each other. They talk through actions
  (`src/platform/actions.ts`), contributions (`src/workbench/contributions.ts`) and
  the services that file exposes (editor, tool windows, notifications).
- `src/editor/` — CodeMirror itself: the model, the LSP and syntax features,
  navigation, refactoring. The backend keeps the authoritative copy of every open
  document; the editor sends change sets tagged with the version they were made
  against.
- `src/ui/` — design-system primitives. Styling is CSS Modules plus the tokens in
  `src/styles/tokens.css`.
- **User-visible text** goes through one of two catalogs in
  `src/platform/locales/`: keyed messages (`en.json`, `zh.json`) for `t()`, and
  source-text catalogs (`text/<locale>.json`) for `tx`-tagged English literals.
  English is the source, so a new string needs no English entry — only the other
  locales'.

## Contribution points outside the build

- `languages/<id>.toml` — one file per language: detection, comments, indentation,
  tree-sitter grammar, language servers, debug adapter. 77 of them today.
- `wit/` — the plugin interface: manifest, providers, views, host functions,
  events, workspace. `crates/ide-extension` validates a package's declarations and
  commits them all or nothing.
- `plugins/official/` — plugins shipped with the IDE (MyBatis navigator, Solidity,
  CSV editor, regex tester, license header, and others). `templates/` holds
  starter packages; `fixtures/` holds sample projects for manual and automated
  testing.
- Themes are plugin contributions: a manifest lists theme files, and the tokens
  they may set are the CSS variables of `src/styles/tokens.css`
  ([themes](plugins/themes.md)).

## Rules that keep it working

- No domain logic in `src-tauri/`; no cross-feature imports in `src/features/`.
- No hex colours in components — tokens only. Real `<button>`/`<input>`, inline SVG
  icons from `src/ui/icons.tsx`, `aria-label` on icon buttons.
- New IPC types belong in the crate that owns them and must derive the specta type,
  then `pnpm gen:bindings` runs and `src/bindings.ts` is committed.
- A command that the remote server also needs is registered in its dispatcher; a
  window-scoped operation takes the window whose project it applies to.

## Where to look next

| You want to change                        | Start at                                                                                    |
| ----------------------------------------- | ------------------------------------------------------------------------------------------- |
| A language's detection, comments, servers | `languages/<id>.toml`                                                                       |
| Editor behaviour, decoration, navigation  | `src/editor/`, `crates/ide-syntax`                                                          |
| Language server lifecycle or features     | `crates/ide-lsp`; installing the server, `crates/ide-tools`                                 |
| Debugging                                 | `crates/ide-dap`                                                                            |
| Run configurations and processes          | `crates/ide-run`, `src/features/run`                                                        |
| Git                                       | `crates/ide-vcs`                                                                            |
| Plugins                                   | `crates/ide-extension`, `crates/ide-plugin`, `wit/`                                         |
| Settings                                  | `crates/ide-core/src/settings.rs`, `src/features/settings`                                  |
| A brand-new feature                       | `src/features/<name>/index.ts`, `crates/ide-<domain>`, `src-tauri/src/commands/<domain>.rs` |
