# AGENTS.md

## Project Overview

**Netscope** is a native macOS desktop app for viewing and analyzing HTTP Archive (HAR) files. It provides a Chrome DevTools-like network inspection experience as a standalone app. Built with Electron 28, React 18, TypeScript 5, and Vite 5.

- **Package name:** `netscope`
- **App ID:** `com.netscope.app`
- **Repository:** https://github.com/Dru89/netscope
- **License:** MIT

## Architecture

Netscope is an Electron app with three process layers:

1. **Main process** (`electron/main.ts`) -- Node.js runtime handling window management, file I/O, native menus, macOS file associations, theme detection, and auto-updates (`electron-updater`). Manages multiple windows via a `Set<BrowserWindow>` and tracks loaded files in a `Map<BrowserWindow, string>`.

2. **Preload script** (`electron/preload.ts`) -- Bridge layer using `contextBridge.exposeInMainWorld` to expose a typed `window.electronAPI` with 6 methods. This is the only communication channel between main and renderer.

3. **Renderer** (`src/`) -- A React 18 SPA bundled by Vite. No direct Node.js access. All file I/O goes through IPC. State is managed entirely with React hooks in `App.tsx` (no external state library). Falls back to browser `FileReader` when `window.electronAPI` is unavailable (dev mode).

Detailed architecture docs are in `docs/architecture.md`.

## Directory Structure

```
electron/           Electron main process + preload script
src/
  components/       React components (WelcomeScreen, Toolbar, RequestTable, DetailPanel, SummaryBar)
  styles/           Plain CSS (global.css for theme variables, app.css for component styles)
  types/            TypeScript types (HAR spec types, electron API declarations)
  utils/            HAR parsing, formatting, and content type classification
  App.tsx           Root component -- all application state lives here
  main.tsx          ReactDOM entry point
build/              Electron-builder resources (icon.icns)
images/             README screenshots and app icon source (netscope.png)
scripts/
  notarize.js       Apple notarization afterSign hook (uses @electron/notarize)
  release.sh        Tag, push, build, sign, notarize, publish to GitHub Releases
site/               Marketing website (Astro 5, deployed to Netlify)
  src/layouts/      Base HTML layout with global CSS
  src/pages/        Single-page landing site (index.astro)
  public/           Static assets (favicons, screenshots)
docs/               Internal docs (architecture.md, features.md, har-format.md)
```

**Build outputs (all git-ignored):** `dist/` (Vite output), `dist-electron/` (bundled main/preload), `release/` (electron-builder .app/.dmg/.zip).

## Code Conventions

- **Components:** Named function exports in PascalCase files. Props interfaces defined inline at the top of each file. Sub-components may be co-located in the same file (e.g., `DetailPanel.tsx` contains 7 components for the panel and its tabs).
- **Types:** HAR spec types prefixed with `Har` (e.g., `HarEntry`, `HarRequest`). App types unprefixed (`FilterState`, `SortState`). Computed/internal fields prefixed with underscore (`_index`, `_url`, `_transferSize`).
- **Imports:** Use `import type { ... }` for type-only imports. A `@/` path alias is configured but not currently used -- existing code uses relative paths.
- **Styling:** Plain CSS with BEM-like class names. Theme via CSS custom properties in `:root` and `@media (prefers-color-scheme: dark)`. No CSS modules, CSS-in-JS, or Tailwind. Color variables follow `--color-{category}-{variant}` naming.
- **State management:** All state in `App.tsx` via `useState`/`useCallback`, passed down as props. Theme preference persisted to `localStorage` key `themeMode`.
- **IPC pattern:** `ipcMain.handle` for renderer-to-main requests; `webContents.send` + `ipcRenderer.on` for main-to-renderer pushes. All listeners return unsubscribe functions for React `useEffect` cleanup.
- **No default exports** except `App.tsx`.

## Key Commands

```bash
npm run dev           # Vite dev server + Electron with hot reload
npm run build         # tsc && vite build && electron-builder (full production build)
npm run build:vite    # tsc && vite build (bundle only, no packaging)
npm run release       # Tag, build, sign, notarize, publish to GitHub Releases
npm run site:dev      # Astro dev server for the marketing site
npm run site:build    # Build the marketing site
```

## Release Process

The `npm run release` script (`scripts/release.sh`):
1. Loads `.env` for Apple signing credentials and `GH_TOKEN`
2. Reads version from `package.json`, creates git tag `v{VERSION}` if needed
3. Pushes commits and tag to origin
4. Runs `tsc && vite build && electron-builder --mac --publish always`
5. electron-builder signs, notarizes (via `scripts/notarize.js`), and uploads DMG + ZIP to GitHub Releases

Required `.env` variables (see `.env.example`): `APPLE_ID`, `APPLE_APP_SPECIFIC_PASSWORD`, `APPLE_TEAM_ID`, `GH_TOKEN`.

## App Icon

The source icon is `images/netscope.png` (2048x2048 RGBA). The macOS `.icns` at `build/icon.icns` is generated from it using `sips` + `iconutil`. Site favicons in `site/public/` are also derived from this source image. If the icon changes, regenerate all derived files.

## Testing

There are no tests in this project. No test framework is configured.

## Important Notes

- The app is **macOS-only** (arm64 targets). electron-builder config only defines `mac` targets.
- `contextIsolation: true` and `nodeIntegration: false` -- the renderer cannot access Node.js APIs directly.
- The marketing site is a separate Astro project in `site/`, deployed to Netlify (configured in `netlify.toml` at the repo root).
- When bumping versions, update `version` in `package.json` and run `npm install --package-lock-only` to sync `package-lock.json`. The release script handles tagging.
