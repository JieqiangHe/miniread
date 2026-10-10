# miniread

A single-file PDF reader with side-by-side AI translation (GLM / DeepSeek with your own key; in the TBtools plugin, TBtools' built-in model too). Live at https://jieqianghe.github.io/miniread/ (GitHub Pages serves `main`). Also packaged as a TBtools plugin (`miniread.plugin`). Forked from an older Chinese 12-provider app and rewritten all-English, minimal.

## Repo rules

- The whole app is one file, `index.html`. The repo mirrors minisvg's layout: `index.html`, `README.md`, `LICENSE` (GPL-3.0, same as minisvg), `.gitignore`, `CLAUDE.md`, `.claude/skills/jh-html-to-tbtools-plugin/` (kept identical to minisvg's copy) and the built `miniread.plugin`. Do not add test files or other folders.
- Plain JavaScript, no framework, no build step. Only external dependency: pdf.js 3.11.174 from cdnjs (viewer + worker); it must keep working as a local/static page. The plugin bundles both files instead (see TBtools plugin).
- Commit to `main` and push to GitHub (remote is SSH: `git@github.com:JieqiangHe/miniread.git`) after each change — standing OK from the user (2026-10-07: everything in the folder goes up, only .DS_Store stays ignored).
- UI text, tooltips, messages and README are English only. Never leave Chinese in the app.
- Code style: dense and short, section comments `// ---------- name ----------`, match the surrounding one-liner style. Don't refactor unasked.
- README is one sentence, the `**Live: ...**` line and the TBtools-II plugin line. Keep it that short.
- UI philosophy: minimal chrome. The user keeps trimming the interface (Help cut to a 5-row shortcut table, prompt textarea hidden, Provider/Model/Think/Key/URL all inside the collapsible "AI settings", language only Simplified Chinese/English, toolbar buttons are plain text). Don't add visible controls or filler text without asking; tuck settings into collapsed `<details>` instead.

## TBtools plugin

- `miniread.plugin` is built by the `jh-html-to-tbtools-plugin` skill in this repo: stage a copy of `index.html` plus `tbtools-ai.json` (`{"capabilities":["chat"]}` — opts the page into the TBtools model bridge), download pdf.js `pdf.min.js` + `pdf.worker.min.js` into `vendor/`, point the script tag and `workerSrc` at relative paths, and load the worker as a classic script too — its `pdfjsWorker` global makes pdf.js 3.11 use the main-thread fake worker, the only mode that works under file://. The repo `index.html` keeps the cdnjs links.
- Rebuild after changing `index.html`. javac lives in the micromamba `openjdk_25.0.2` env (`.../envs/openjdk_25.0.2/lib/jvm/bin` — conda-forge puts the JDK under `lib/jvm`); the TBtools main jar is `~/.TBtools/TBtools_JRE1.6.jar`. Verify with the skill's `Verify.java` (expect `OK: reached WebGuiJPanel`).
- TBtools caveats: the plugin's localStorage sits in TBtools' shared `.jxbrowser` dir — every file:// plugin page can read the stored API keys; dragging PDFs in from Finder is unverified in OFF_SCREEN mode (Open button is the reliable path).

## Architecture (names in index.html)

- Helpers: `$(id)`, `V` = `#mView`, `ls(k,v)` = localStorage get/set, `el/btn/msg/nl`.
- AI config: `AI` map (provider → `[default URL, ...models]`); per-provider storage `mrd<Glm|Ds><Key|Model|Ep>`, global `mrdAi`, `mrdThink`, `mrdLang`, `mrdPrompt`, `mrdAuto`, `mrdThumbs`, `mrdTheme`. `aiLoad()` refills Model/Key/URL on provider switch. `tbAi()` feature-detects the TBtools model bridge; when present it adds the `Tb` provider (first, auto-selected) and `aiLoad()` hides the Model/Think/Key/URL rows for it — in a plain browser nothing changes.
- `translate(go)`: no argument (button click) acts as Stop when a request is running; a truthy argument (Ctrl/⌘+Enter, auto-selection) aborts the previous request and starts a new one. Ownership token `myAc`/`ok()` keeps a superseded run from clobbering `ac`, the button label or `msg`. Bridge runs (`br`) use `tb.chat` with `onDelta` (whole text so far), abort via `requestId`, map rejection codes through `tbWhy`; single message capped at 32,000 chars.
- SSE stream: `delta.reasoning_content` feeds a collapsed thinking note, `delta.content` the output; errors from `j.error` and empty replies are surfaced.
- PDF: `open/openPdf/layout/draw/zoom/fit/goto/cur`. `gen` cancels stale layouts; `layP` is the last layout promise (`goto` retries through it when the page div isn't built yet); pages lazy-render via IntersectionObserver (800px margin), thumbnails (96px wide) via `tgen/tobs/tdraw`. The `--scale-factor` CSS var must stay on `.page` (pdf.js text layer needs it). Scale is clamped to 0.1–8; Ctrl-wheel zoom is debounced 60 ms (`wzT`).
- Keys: `?` help, `+`/`-`/`0` zoom, `←`/`→` page step, `↑`/`↓`/`Space` scroll, `Home`/`End` first/last. The handler skips events targeted at inputs/buttons.
- Selection: `mouseup` on the viewer → dehyphenate `-\n` before lowercase → collapse whitespace → `lastSel` dedupe → auto-translate if `#mAuto`. Drag&drop is handled at document level and only for files, so text drops into the textarea still work.
- Actions: `data-a` buttons mapped in the object `A`; theme via `setTheme` + `mrdTheme`, with an inline `<script>` applying it before paint.

## Pitfalls

- `#mRight details label{display:grid}` beats the UA `[hidden]` rule; hidden panel rows rely on `#mRight [hidden],#mRight details label.up[hidden]{display:none}`. Don't remove that.
- The word-count line contains literal `\uXXXX` ranges; those can't be matched through the Edit tool (escapes get decoded). Edit that line via python or split edits around it.
- pdf.js is pinned to 3.11.174: v4 replaced `pdfjsLib.renderTextLayer` with a `TextLayer` class — don't upgrade casually.
- DeepSeek receives both `thinking:{type:'enabled'}` and `reasoning_effort`; fine with the current models, revisit if the API rejects them.

## Testing

- No test suite. Syntax-check every inline `<script>` block: extract with python, then `new Function(src)` via `osascript -l JavaScript` (no node/deno/bun on this machine).
- Static checks with python: id uniqueness, every `$()` reference has an element, every `data-a` button has an `A` entry.
- Visual check: `open index.html`, then ⌘R to reload (the user reviews screenshots; `open` alone does not reload an existing tab).
