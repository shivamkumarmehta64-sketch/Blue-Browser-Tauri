# Blue Browser (Tauri)

[![CI](https://github.com/shivamkumarmehta64-sketch/Blue-Browser-Tauri/actions/workflows/ci.yml/badge.svg)](https://github.com/shivamkumarmehta64-sketch/Blue-Browser-Tauri/actions/workflows/ci.yml)

> **Cross-platform desktop browser engineered in Rust, Tauri v2 (`2.11.5`), vanilla TypeScript (`5.6.2`), and Vite (`6.0.3`) — pairing native system webviews with a built-in Brave ad-blocking engine and embedded MCP server.**  
> Companion project to [Black Browser (`BlackBrowser`)](https://github.com/shivamkumarmehta64-sketch/BlackBrowser) · [3D Portfolio Entry](https://shivamkumarmehta64-sketch.github.io/all-websites-3d/#blue-browser-tauri).

---

## Architecture

Blue Browser separates a lightweight, dependency-free vanilla TypeScript/CSS UI shell from a Rust/Tauri v2 backend responsible for child-webview tab orchestration, network & cosmetic ad blocking, persistent storage, CORS-free autocomplete, and local MCP automation.

### Rust Backend (`src-tauri/src/`)

- **`lib.rs` — Tab Engine, Storage & Bridges:**
  - **Child-Webview `TabHost`:** Attaches native `WebviewBuilder` child webviews directly onto the frameless `main` window (`open_url`, `set_tab_bounds`, `activate_tab`, `close_tab`), enforcing `http(s)://` navigation and intercepting `_blank` / `window.open` targets into `new-tab-request` events.
  - **SPA Route Tracking:** Injects an initialization script wrapping `history.pushState`, `history.replaceState`, `popstate`, and `hashchange` to sync page URL/title changes back to `set_page_info` (verified against `tab-<id>` webview labels).
  - **Persistent JSON `Store`:** Stores per-key JSON blobs (`settings`, `quicklinks`, `bookmarks`, `reading`, `history`, `notes`, `session`) inside `app.path().app_data_dir()`, plus an aggregated `home_data` command for the New Tab Page.
  - **Async JS Eval & Find-in-Page Bridge:** `eval_in_tab` / `eval_in_tab_str` (`mpsc` channel with 10s timeout) powers Reader Mode article extraction and per-tab zoom; `find_in_page` highlights matches via TreeWalker `<mark data-find>` traversal.
  - **Search Suggestions Proxy:** Uses `ureq` (`2.12.1`) from Rust to fetch Google, Bing, and DuckDuckGo autocomplete suggestions without hitting WebView CORS restrictions.
- **`adblock.rs` — Brave Adblock & Cosmetic CSS Server:**
  - Wraps Brave's `adblock` (`0.13.2`) `Engine` behind `RwLock`, initialized from `src-tauri/assets/filters/bundled_filters.txt`.
  - Tracks session `ads_blocked` and `trackers_blocked` atomic counters with bounded network (`1024`) and cosmetic (`512`) caches and live shields toggling (`set_shields`, `shields_status`, `privacy_stats`).
  - Spawns a `tiny_http` (`0.12.0`) loopback server on `127.0.0.1:0` (`spawn_cosmetic_server`) serving per-URL element-hiding stylesheets at `/adblock.css?url=...` with CORS headers for document-start injection into child tabs.
- **`mcp.rs` — Embedded MCP Server (`blue-browser-mcp`):**
  - Launches a background stdio JSON-RPC 2.0 server (`protocolVersion: "2024-11-05"`) during Tauri `.setup(...)`, exposing `browser_navigate`, `browser_new_tab`, and `browser_get_tabs` so external AI agents/clients can drive the browser directly.

### Frontend Shell (`index.html`, `src/main.ts`, `src/styles.css`)

- **Frameless Glassmorphic UI:** Custom window titlebar and controls (`1280×820`, min `920×600`, `decorations: false`).
- **Tab Management:** Horizontal and vertical tab layouts, drag-to-reorder, pinned tabs, mute toggle, private/incognito tabs, middle-click close, and reopen closed tab (`Ctrl+Shift+T`).
- **Omnibox & Navigation:** Security indicator badge, frecency-ranked local bookmarks/history combined with backend search suggestions, per-tab zoom (`25%`–`500%`), Find-in-Page (`Ctrl+F`), and Reader Mode overlay.
- **Command Palette & Side-Rail Panels:** Keyboard-driven Command Palette (`Ctrl+Space` / `F1`), New Tab Page (greeting, quick links, recent history, stats), and slide-out panels for Bookmarks, History, Reading List, Notes, Downloads, Privacy & Shields, and Settings.
- **Theme Engine:** `system`, `light`, and `dark` themes, 6 accent color presets, and adjustable UI scaling (`75%`–`140%`).

---

## Repository Structure

```text
Blue-Browser-Tauri/
├── index.html                  # Frameless browser shell, panels, overlays & modals
├── package.json                # Vite 6 + TypeScript 5.6 + @tauri-apps/api/cli v2
├── src/
│   ├── main.ts                 # Tab manager, omnibox frecency, command palette, IPC & UI logic
│   └── styles.css              # Glassmorphic themes, vertical/horizontal tabs, rail & panels
├── src-tauri/
│   ├── Cargo.toml / Cargo.lock # Rust crate manifest (tauri 2.11.5, adblock 0.13.2, tiny_http, ureq)
│   ├── tauri.conf.json         # Window, CSP, build hooks, and NSIS bundle config
│   ├── assets/filters/
│   │   └── bundled_filters.txt # Offline baseline EasyList/EasyPrivacy filter rules
│   ├── capabilities/
│   │   └── default.json        # Tauri v2 window, opener & event permissions
│   └── src/
│       ├── main.rs             # Binary entry point
│       ├── lib.rs              # TabHost, Store, search proxy, JS eval & Tauri commands
│       ├── adblock.rs          # Brave AdblockManager, loopback cosmetic server & unit tests
│       └── mcp.rs              # Embedded stdio JSON-RPC 2.0 MCP server
└── .github/workflows/ci.yml    # GitHub Actions CI (frontend build, cargo check, cargo test)
```

---

## Prerequisites

- **Node.js** `20+` (CI uses Node `22`) and `npm`
- **Rust** stable toolchain (`rustup` + `cargo`)
- **Tauri v2 OS dependencies:**
  - **Windows:** Microsoft Edge WebView2 Runtime + MSVC Build Tools (`C++`)
  - **Linux (Debian/Ubuntu):** `libwebkit2gtk-4.1-dev`, `libgtk-3-dev`, `libayatana-appindicator3-dev`, `librsvg2-dev`, `libxdo-dev`, `libssl-dev`, `patchelf`
  - **macOS:** Xcode Command Line Tools (`xcode-select --install`)

---

## Build, Run & Test

### 1. Install Dependencies & Run in Development
```bash
git clone https://github.com/shivamkumarmehta64-sketch/Blue-Browser-Tauri.git
cd Blue-Browser-Tauri
npm install
npm run tauri dev
```

### 2. Production Build
- **Frontend bundle only (`tsc && vite build` -> `dist/`):**
  ```bash
  npm run build
  ```
- **Full Tauri installer / release bundle:**
  ```bash
  cargo tauri build
  # or via npm scripts:
  npm run build:release
  ```
- **Executable only without installer bundle (`--no-bundle`):**
  ```bash
  npm run build:exe
  ```

### 3. Local Verification & Unit Tests
`tauri.conf.json` configures `bundle.resources` to package `target/release/WebView2Loader.dll` for NSIS releases, and `tauri::generate_context!()` expects `dist/` to exist. Build the frontend first and override `TAURI_CONFIG` during debug checks/tests:

- **bash / zsh (macOS & Linux):**
  ```bash
  npm run build
  cd src-tauri
  TAURI_CONFIG='{"bundle":{"resources":null}}' cargo check --locked
  TAURI_CONFIG='{"bundle":{"resources":null}}' cargo test --locked --lib
  ```
- **PowerShell (Windows):**
  ```powershell
  npm run build
  cd src-tauri
  $env:TAURI_CONFIG = '{"bundle":{"resources":null}}'
  cargo check --locked
  cargo test --locked --lib
  ```

---

## Downloads

Pre-built Windows installers (`.exe` & `.msi`) are available on the [Releases page](https://github.com/shivamkumarmehta64-sketch/Blue-Browser-Tauri/releases/tag/v0.1.0).

## License

MIT License — see [`LICENSE`](./LICENSE).