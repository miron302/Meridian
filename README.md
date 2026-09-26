# Meridian

A desktop browser built on Chromium, with a Liquid Glass
interface, a real network-level ad/tracker blocker, and support for
installing Chrome extensions.

---

## Contents

- [Quick start](#quick-start)
- [Architecture](#architecture)
- [What's real vs. what's a documented limitation](#whats-real-vs-what-is-a-documented-limitation)
- [Ad blocking](#ad-blocking)
- [Extensions](#extensions)
- [Keyboard shortcuts](#keyboard-shortcuts)
- [Building a distributable app](#building-a-distributable-app)
- [Manual test checklist](#manual-test-checklist)
- [Project layout](#project-layout)
- [License](#license)

---

## Quick Start
Download the latest version from releases.

---

## Building

Requires **Node.js 18+** and **npm**. Tested with Electron 35 on macOS 14+,
and should run on Windows/Linux via the same codebase (see platform notes
below).

```bash
git clone https://github.com/miron302/Meridian
cd meridian
npm install
npm run dev       
# or
npm start           
```

On first launch, Meridian creates its user-data directory (settings,
bookmarks, history, downloads metadata, and the ad-blocker's compiled
filter-list cache) under the OS-standard Electron `userData` path
(`~/Library/Application Support/Meridian` on macOS).

## Architecture

```
src/
  main/                   # Node/Electron main process
    main.js               # app lifecycle, security policy, wiring
    window-manager.js      # creates BrowserWindows (glass chrome + tabs)
    tab-manager.js          # one Chromium BrowserView per tab
    menu.js                 # native OS menu + shortcuts
    internal-protocol.js    # the meridian:// scheme (new tab, settings, ...)
    downloads-manager.js    # wraps session 'will-download'
    constants.js             # IPC channel names, defaults, filter-list URLs
    adblock/adblock-manager.js     # network-layer ad/tracker blocking
    extensions/extension-manager.js # real Chrome extension loading
    store/                  # electron-store-backed persistence
    ipc/ipc-handlers.js      # the only place renderer requests are handled
  preload/
    preload.js               # privileged bridge for Meridian's own chrome
    webview-preload.js        # bridge for tab content, gated to meridian://
  renderer/
    index.html, js/app.js, styles/   # the glass tab bar + toolbar
    pages/{newtab,settings,extensions,history,bookmarks,downloads}/
```

**Security model:** every renderer (the chrome window and every tab) runs
with `contextIsolation: true` and `sandbox: true`. Nothing gets Node
integration. The *only* privileged surface is the `window.meridian` API
installed by the preload scripts, which is a fixed, auditable set of
`ipcRenderer.invoke` calls into `src/main/ipc/ipc-handlers.js` — there is no
generic "run arbitrary main-process code" channel. Regular web content
loaded in a tab gets **no** privileged API at all: `webview-preload.js` only
installs the bridge when `location.protocol === 'meridian:'`, a custom
scheme that only Meridian's own bundled pages can ever be served from (see
`internal-protocol.js`). `webSecurity` is never disabled anywhere.

**Why Electron/BrowserView, not CEF:** Electron was chosen because it gives
first-class, actively maintained Node ⇄ Chromium IPC, an ergonomic native
menu/window API, and — critically for the extension requirement — a real,
built-in Chrome-extension loader (`session.loadExtension`). CEF would need a
custom extension subsystem built from scratch. Each tab is an Electron
`BrowserView` layered under the HTML chrome (see `TOOLBAR_HEIGHT` in
`tab-manager.js`), giving genuine per-tab process isolation and independent
back/forward history — not an iframe.

## What's real vs. what is a documented limitation

This project follows one rule throughout: if something can't be done for
real with the chosen framework, it's implemented as the closest honest
approximation and clearly commented — never faked.

| Feature | Status |
|---|---|
| Tabs, navigation, history, bookmarks, downloads | Real. Backed by actual Chromium `BrowserView`s and Electron's `session`/download APIs. |
| Ad/tracker blocking | Real, network-layer. `@ghostery/adblocker-electron` compiles EasyList/EasyPrivacy/uBlock filter lists and hooks `session.webRequest.onBeforeRequest`/`onHeadersReceived` directly — matching requests are cancelled before the response body downloads. Not a content-script `div.ad { display:none }` hack. |
| Cosmetic filtering | Real (CSS-hiding rules from the same filter lists), toggle in Settings. |
| Extension loading (unpacked + `.crx`) | Real. Uses `session.loadExtension`; content scripts, background workers, `chrome.storage`, messaging, and extension pages all run under Chromium's actual extension implementation. |
| Extension toolbar icons / popups | **Approximated by design.** Electron does not render a `chrome.action` toolbar UI the way Chrome does. Meridian reads each extension's manifest itself, draws its own toolbar button, and opens the extension's real `default_popup`/`options_page` (a genuine `chrome-extension://` page with full extension APIs) in a small window it manages. See the comment block at the top of `extension-manager.js`. |
| Manifest V3 edge cases | Partial, inherited from Electron/Chromium's own MV3 support level (varies by API — e.g. some `declarativeNetRequest` dynamic-rule and `chrome.scripting` edge cases may not fully match Chrome). Not something this app can paper over. |
| Incognito windows | Real. Uses a fresh **non-persistent** Electron session per incognito window — cookies/cache/localStorage never touch disk. |
| macOS vibrancy / traffic lights / hidden-inset title bar | Real native Electron/macOS APIs (`vibrancy`, `titleBarStyle: 'hiddenInset'`). |
| Site permission prompts (camera/mic/geo/notifications) | Real, backed by `session.setPermissionRequestHandler`, default-deny until the user grants per-site. |
| Password/credential secure storage | **Not implemented in this pass.** Electron's `safeStorage` API (OS Keychain/DPAPI-backed) is the correct integration point; wiring a full autofill/credential-manager UI was out of scope for this build. |
| Windows/Linux builds | The same codebase targets them via `electron-builder` (see `package.json`), but macOS is the platform this build was designed and verified against; Windows/Linux-specific polish (e.g. non-vibrancy chrome fallback) may need a pass. |

## Ad blocking

Enabled by default. On first launch, `AdblockManager` fetches and compiles:

- `easylist.to/easylist/easylist.txt`
- `easylist.to/easylist/easyprivacy.txt`
- uBlock Origin's `filters.txt` and `privacy.txt` (from the `uAssets` repo)

into a binary engine cached under the app's `userData/adblock-cache`
directory, then hooks it into Chromium's `webRequest` API for every session
(including each incognito window's own session). Settings → Ad Blocking
lets you re-fetch the lists on demand. The shield icon in the toolbar shows
live per-site ad/tracker counts and lets you add a per-site exception
(which disables blocking for that site's session and reloads the page).

> **Sandboxed test environments:** if you're running this inside a
> network-restricted sandbox/CI box that blocks arbitrary outbound HTTPS,
> the initial filter-list fetch will fail and the blocker will simply run
> with an empty rule set rather than crashing — this was confirmed during
> development. On a normal machine with regular internet access, the fetch
> succeeds immediately.

## Extensions

Go to `meridian://extensions` (or Menu → Extensions):

- **Load unpacked** — pick a folder containing a `manifest.json`. This is
  Chromium's real developer-mode unpacked-extension loader.
- **Install .crx** — pick a `.crx` file; Meridian strips the CRX header,
  unzips the payload, and loads it the same way as an unpacked extension.
- Enable/disable/remove, and view each extension's declared permissions.
- If an extension has a `default_popup`, its icon appears in the toolbar;
  clicking it opens the real popup page. If it only has an options page,
  clicking opens that instead.

## Keyboard shortcuts

| Action | macOS | Windows/Linux |
|---|---|---|
| New tab | ⌘T | Ctrl+T |
| New window | ⌘N | Ctrl+N |
| New incognito window | ⌘⇧N | Ctrl+Shift+N |
| Close tab | ⌘W | Ctrl+W |
| Reopen closed tab | ⌘⇧T | Ctrl+Shift+T |
| Reload | ⌘R | Ctrl+R |
| Focus address bar | ⌘L | Ctrl+L |
| Find in page | ⌘F | Ctrl+F |
| Bookmark page | ⌘D | Ctrl+D |
| Show bookmarks | ⌘J | Ctrl+J |
| Show history | ⌘H | Ctrl+H |
| Next/previous tab | ⌘⌥Tab / ⌘⇧Tab* | Ctrl+Tab / Ctrl+Shift+Tab |
| Back / Forward | ⌘\[ / ⌘\] | Ctrl+\[ / Ctrl+\] |

\* Tab cycling is handled in the renderer (`app.js`) via Cmd/Ctrl+Tab; all
other shortcuts are native OS menu accelerators (`menu.js`), so they work
even when the window doesn't have focus on the chrome layer.



## License

MIT — see [LICENSE](LICENSE). Bundled/depended-on third-party components
keep their own licenses (Electron/Chromium, `@ghostery/adblocker-electron`,
etc.) — see LICENSE for details. Default filter lists are fetched at
runtime from their upstream maintainers, not redistributed in this repo.
