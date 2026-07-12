# Designing a macOS Code Editor in Swift

Planning notes for building a native macOS code editor (project scaffold:
`~/Developer/SwiftCodeEditor`, full plan in its `docs/PLAN.md`) — architecture
choices, effort estimates, the risk register, and the workspace-based vs
document-based app decision.

Requirements: local Git repo support (no remotes); nine languages (Swift,
Python, C++, Java, JavaScript, Markdown, HTML, LaTeX, XML); repo-wide search;
navigation (go to definition, find usages); refactoring (rename, change
signature). Targets macOS 26+ (Tahoe), Swift 6.3+.

---

## Core strategy: don't build language intelligence in-house

Building go-to-definition or rename natively for nine languages is a
multi-year project. Instead the editor is a thin, well-crafted client over
proven engines:

| Layer | Technology | Why |
|---|---|---|
| App shell | SwiftUI (chrome, panels, settings) | Fast iteration; macOS 26 floor means no fallback paths |
| Editor view | AppKit `NSTextView` on **TextKit 2** | SwiftUI has no viable code-editor text engine; TextKit 2 gives viewport-based layout for large files |
| Highlighting | **tree-sitter** (SwiftTreeSitter) | Incremental parsing; mature grammars for all nine languages |
| Navigation & refactoring | **LSP client** (JSON-RPC over stdio) | Definition, references, rename, diagnostics, completion per language via existing servers |
| Git | **libgit2** (SwiftGit2) | Local-only requirement removes all auth/credential work — roughly half of typical Git-integration effort |
| Repo search | Bundled **ripgrep** | Battle-tested, gitignore-aware, streams JSON results |

Per-language servers: sourcekit-lsp (Swift), Pyright or pylsp+rope (Python),
clangd (C++), Eclipse JDT LS (Java), typescript-language-server (JS),
vscode-html-language-server, texlab (LaTeX), lemminx (XML), marksman
(Markdown, optional). JDT LS and lemminx share one bundled JRE; two servers
need Node.

## Decision: workspace-based, not document-based

Apple's document-based app template (`NSDocument` / `DocumentGroup`) makes
**one file the unit of everything** — window per document, per-document undo,
autosave-in-place, Versions, free open/save/recents plumbing. That fits
TextEdit or Pages, where a user works on one self-contained file.

A code editor's real "document" is the **repo folder**. One window represents
a workspace; many file buffers open in tabs inside it; and long-lived
services — LSP servers, the Git index, search, the FSEvents watcher — attach
to the workspace, not to any file. Forcing pure NSDocument onto this fights
the framework:

- Window-per-document is backwards for a tabbed, one-window-per-repo UI.
- Autosave/Versions are per file, but branch switches and cross-file renames
  change many files at once, outside any single document's lifecycle;
  atomic multi-file undo doesn't map onto per-file undo managers.
- Shared workspace services have no home in the NSDocument model.

The cost: dirty tracking, save prompts, per-buffer undo managers, external
change detection, and disk-conflict handling must be built by hand.

**Chosen hybrid (the Xcode / CodeEdit pattern):** make the *workspace itself*
a single `NSDocument` subclass whose "file" is the folder, with read/write
overridden to no-ops. That buys the cheap lifecycle chrome — Open Recent
listing repos, window restoration, open-panel and Finder/dock integration —
while file buffers underneath stay plain model objects owned by the
workspace layer.

## Phases & estimates (1–2 devs, engineer-weeks)

| Phase | Scope | Effort |
|---|---|---|
| 0 | Foundation: workspace/document model, chrome, file tree, FSEvents | 3 |
| 1 | Editor core: TextKit 2 view, tree-sitter pipeline, folding, perf | 8–10 |
| 2 | Search: ripgrep integration, multi-file replace with preview | 3 |
| 3 | Git: status, staging, commits, branches, history, diffs | 4–5 |
| 4 | LSP infrastructure: transport, server lifecycle, diagnostics, completion | 6–8 |
| 5 | Navigation: go to definition, find usages, symbol search | 2–3 |
| 6 | Refactoring: LSP rename with atomic cross-file undo; change signature where supported | 3–4 |
| 7 | Hardening: settings, notarization, Sparkle, QA across nine languages | 5–6 |

Total ~34–42 ew; **~41–50 ew with a 20% buffer** ≈ 10–12 months for one dev,
5–7 months for two. Dogfood milestone: Phases 0–2 plus minimal Phase 4
(~4–5 months solo) yields a daily-driver editor with highlighting, search,
and diagnostics.

## Key Principles

- **"Change signature" is the requirement trap.** Rename is standard LSP
  (`textDocument/rename`) and works everywhere; change signature is *not in
  the LSP spec*. Only Eclipse JDT LS (Java) and rope (Python) offer it.
  Scope it to those two or budget 6–12 extra months for per-language
  refactoring engines.
- **Editor performance can't be retrofitted.** Profile the TextKit 2 +
  tree-sitter pipeline from day one against 50k-line files; a spike on this
  is the first implementation step, including evaluating the MIT-licensed
  `CodeEditSourceEditor` as a shortcut (potentially −4–6 ew).
- **LSP server ops are real product surface.** Eight external processes with
  three runtimes (JRE, Node, native) need version management, crash-restart,
  and per-project config (`compile_commands.json` for clangd; sourcekit-lsp
  prefers SwiftPM packages — Xcode-project codebases need
  `xcode-build-server`).
- **Distribute outside the Mac App Store.** Bundled helper binaries and
  spawned language servers make sandboxing impractical; plan for
  notarization + Sparkle instead.
- **Local-only Git is a huge simplification.** No push/pull/fetch means no
  credential UI, SSH agents, or keychain integration.
- **Module boundaries mirror the phases** (SwiftPM targets in the scaffold):
  `App`, `WorkspaceCore`, `EditorCore`, `SyntaxKit`, `SearchCore`, `GitCore`,
  `LanguageIntelligence` — each stub documents its phase and planned
  components, with dependencies commented in `Package.swift` until their
  phase begins.
