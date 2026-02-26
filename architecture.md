# Architecture

## Runtime Topology

Four execution contexts:

### 1. Background Page (control plane)
- Entry: `background/background.html` → `background/background.js` (2.9MB minified)
- Also loads: `background/health-check.js` (responds to `health-check-request`)
- Central broker for all policy decisions, vault operations, native messaging, sync, telemetry
- Initializes: database, core interface (WASM), feature flags (Unleash), native app connection, XAM backend, context menus, Watchtower data
- Event subscriptions handle account changes, lock/unlock, session state transitions
- Exports `b5xHandlers` for b5 web app integration and `initializeFinishedPromise` for startup gating

### 2. Content Script Bootstrap (every page, every frame)
- Entry: `inline/inject-content-scripts.js` at `document_start`, `all_frames: true`, `<all_urls>`
- Guards against double-injection (`injectJsHasStarted` property)
- Dynamically imports two modules:
  - `/inline/injected.js` (368KB) — always loaded (page managers, autofill, inline menu, frame management)
  - `/inline/injected/heuristics.js` — conditionally loaded when `login-detection-is-enabled` returns true from background
- Import retry logic: 3 attempts with 25ms/50ms delays between retries
- Error reporting via `report-error` message to background
- Initializes a `LogReporter` (logger) that forwards all content-script logs to background via `new-tab-log-event`

### 3. Specialized Content Scripts (host-specific)
Declared in manifest, loaded on matching hosts:

| Script | Hosts | Timing | Purpose |
|--------|-------|--------|---------|
| `webauthn.js` + `webauthn-listeners.js` | `https://*/*`, `http://localhost/*` | `document_start` | WebAuthn/passkey mediation (see [webauthn-analysis.md](webauthn-analysis.md)) |
| `b5.js` | `*.1password.com/ca/eu`, `*.b5dev.*`, `*.b5test.*`, `*.b5local.*`, `*.b5staging.*`, `*.b5rev.*` | `document_idle` | 1Password web app integration, SSO completion, session init |
| `kolide.js` | `app.kolide.com/ca/eu`, `auth.kolide.com/ca/eu` | `document_start` | Kolide device trust / EPM integration |
| `secure-remote-autofill-start-pairing.js` | `www.director.ai/?*`, `www.director.ai/` | `document_end` | Remote autofill pairing initiation |
| `secure-remote-autofill-complete-pairing.js` | `www.director.ai/complete-1password-pairing*` | `document_end` | Remote autofill pairing completion |
| `autofill.js` | `autofill.me/*` | `document_start` | Test/demo autofill site |

### 4. Extension UI Surfaces
- `app/app.html` + `app/app.js` (682KB) — main extension window/panel
- `popup/index.html` + `popup/index.js` (399KB) + `popup/set-popup-width.js` — browser action popup
- `launcher/apps.html` + `launcher/apps.js` — app launcher
- `inline/menu/menu.html` — inline autofill suggestion menu (web-accessible)
- `inline/modal/modal.html` — modal dialogs for Privacy.com, email alias, Brex (web-accessible)
- `inline/notification/notification.html` — save/update notifications (web-accessible)
- `inline/universal-sign-on/universal-sign-on.html` — USO banner (web-accessible)
- `inline/tutorial/tutorial.html` — onboarding tutorial
- `devtools/devtools.html` + `devtools/panels.html` — DevTools logging panel

## Chunk System

356 chunk files under `chunks/`. Two categories:
- **Code chunks**: `chunk-{HASH}.js` — shared logic modules (largest: `chunk-OJR52IF5.js` at 1.9MB, `chunk-MSIWLBOQ.js` at 804KB, `chunk-22IBMJDR.js` at 138KB)
- **Icon/asset chunks**: named by icon (e.g., `icon_creditcard_color_32-HASH.js`, `sso_login_okta_32-HASH.js`)

Semantic chunk name patterns observed: `account-family`, `account-team`, `developer_watchtower`, `browserbase_logo`, `browseruse-icon`, `anchor-browser-icon`, `browser-polyfill`, `import_guide_pen`.

## WASM Modules

Seven WebAssembly modules in `assets/wasm/` (total ~30MB):

| Module | Size | Likely Purpose |
|--------|------|----------------|
| `op_wasm_b5x_bg` | 15.6MB | Core vault/crypto operations for b5x (extension) |
| `op_wasm_xam_bg` | 11.2MB | XAM (cross-app management / device trust) backend |
| `confidential_computing_bg` | 1.8MB | Confidential computing primitives |
| `b5_trustlog_bg` | 1.3MB | Trust log generation/verification |
| `b5_trust-verifier_bg` | 1.0MB | Trust verification |
| `b5_mycelium_bg` | 319KB | Mycelium relay protocol (P2P communication for remote autofill) |
| `b5_hpke_bg` | 82KB | Hybrid Public Key Encryption (RFC 9180) |

The background.js initialization calls `rA.init(e)` ("initializeCoreInterface") which loads the main WASM module. CSP allows `wasm-unsafe-eval` for this purpose.

## Permission Profile

### Always granted
`<all_urls>`, `alarms`, `clipboardWrite`, `contextMenus`, `downloads`, `idle`, `management`, `nativeMessaging`, `notifications`, `privacy`, `scripting`, `storage`, `tabs`, `webNavigation`, `webRequest`, `webRequestBlocking`, `declarativeNetRequestWithHostAccess`

### Optional
`bookmarks`

### Web-Accessible Resources
Source maps (`*.js.map`), fonts, images, and critically: `inline/injected.js`, `inline/injected/heuristics.js`, `inline/injected/styles/inline-tooltip.css`, and all inline UI HTML files (menu, notification, modal, universal-sign-on). These can be loaded/detected by any web page.

## Feature Flags

Unleash-based feature flag system with two tiers:
- **Pre-registration flags**: evaluated before any account is signed in (e.g., `b5x-pre-auth-tracing`)
- **Account-gated feature trials**: per-account feature flags from server

Background broadcasts `unleash-features-changed` events to all listeners when flags update. Content scripts query individual flags (e.g., `login-detection-is-enabled`).

## Initialization Sequence

From `background.js` main init function (reconstructed):

1. Initialize storage
2. Initialize feature flags cache
3. Check for terminated DB, set icon
4. Get browser language
5. Initialize Sentry
6. Init build info
7. Initialize core interface (WASM load)
8. Set locale
9. Initialize database (IndexedDB)
10. Get device info, sync pre-registration feature flags
11. Start performance observer (if tracing flag set)
12. Load all accounts, set up account handlers
13. Initialize crypto
14. Subscribe to events: session changes, account updates, lock/unlock
15. Load Watchtower data
16. Initialize native app connection
17. Initialize unlock-with-context cache
18. Enable insiders (if appropriate)
19. Initialize XAM backend
20. Initialize App Launcher
21. Migrate storage (if needed)
22. Initialize context menus
23. Initialize notifications

## External Extension Communication

3 references to `chrome.runtime.onMessageExternal` — the extension accepts messages from other extensions (likely 1Password desktop app or enterprise connectors). No `externally_connectable` manifest key found, so Firefox's default policy applies.

## Build/Signing

- Build channel: `stable`
- Signed by Mozilla AMO Production Signing Service
- COSE + RSA signatures in `META-INF/`
- Sentry debug IDs embedded in every JS file for crash correlation
