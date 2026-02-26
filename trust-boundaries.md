# Trust Boundaries & Dataflow

## Trust Zones

### Zone A — Background (highest extension privilege)
- Full access to all `chrome.*` APIs
- Holds vault state, account sessions, crypto keys **in both JS heap and WASM memory**
- Master Unlock Key (MUK) stored as exportable JWK on account handler objects in JS
- SRP-X cached in JS on client context objects
- Decrypted item secrets (passwords, OTPs, card numbers) transit through JS heap during fill operations
- Central policy decision point for all sensitive operations
- Only context that communicates with native host and remote services

### Zone B — Content Scripts (extension context in page)
- Runs in isolated world alongside untrusted page DOM
- Can read/modify page DOM but shares no JS state with page
- Communicates with background via `chrome.runtime.sendMessage`
- Communicates with page world via `window.postMessage` (WebAuthn only)

### Zone C — Page World (untrusted)
- `webauthn-listeners.js` runs here — only extension code in this zone
- Fully hostile environment — page JS can observe, intercept, modify anything
- Communication with extension only via `window.postMessage` protocol

### Zone D — Native Host (desktop app)
- Connected via `nativeMessaging` permission
- `browser.runtime.sendNativeMessage("")` — empty string as native app ID (Firefox uses the manifest `applications.gecko.id` for routing)
- Native app connection initialized during background startup (`initializeNativeAppConnection`)
- **Confirmed operations via native messaging:**
  - Biometric unlock: save/retrieve/remove MUK + SRP-X from OS secure enclave
  - Biometry availability check
  - dSecret proxy for MFA bypass on trusted devices
  - Device trust public key signing
  - Delegated session management (upgrade from offline state)
  - Desktop connection manager state
- Messages use JSON envelope: `{name: "core", data: JSON.stringify({type: "Biometry", data: {...}})}`
- Trust level: higher than extension — has access to OS keychain/secure enclave

### Zone E — Remote Services (1Password cloud)
- Extensive `connect-src` allowlist (see below)
- WebSocket connections for real-time sync (`wss://b5n.*`)
- REST APIs for account management, vault operations

### Zone F — WASM Modules (portability layer, NOT a security boundary)
- 7 WASM modules loaded with `wasm-unsafe-eval`, compiled from Rust via `wasm-bindgen`
- Provides the same Rust core used by desktop/mobile/CLI clients — portability, not isolation
- **Key material DOES cross the WASM→JS boundary**: MUK exported as JWK, SRP-X cached in JS, decrypted passwords returned to JS for fill
- JS can read WASM linear memory — no privilege separation exists
- WASM's value is **correctness** (battle-tested Rust crypto) not **isolation**
- Only 4 `crypto.subtle.*` calls in background.js; WASM handles virtually all crypto

### Zone G — External Extensions
- `chrome.runtime.onMessageExternal` (3 references)
- Other extensions can send messages to 1Password
- No `externally_connectable` in manifest — on Firefox, any extension can message

### Zone H — Inline UI Frames
- `inline/menu/menu.html`, `inline/modal/modal.html`, etc.
- Web-accessible resources loaded in iframes within web pages
- Have extension origin but are embedded in untrusted page context
- Communicate with background via `chrome.runtime.sendMessage`

## Boundary Crossings

### B0: Page DOM → Content Script
- Content script reads form fields, detects login pages, captures credential data for save
- All DOM-derived data is untrusted input
- Heuristics module classifies pages/forms

### B1: Content Script → Background
- `chrome.runtime.sendMessage` with named message handlers
- ~50 registered handlers in background's `m5({...})` router
- **Critical question: does background validate message schemas?** (needs deep reverse engineering)

### B2: Page World → Content Script (via postMessage)
- WebAuthn IPC protocol (SYN/ACK/Direct)
- Custom message validation (type/source/name/msgId checks)
- `stopImmediatePropagation` on matched messages
- **Risk: any page script can craft these messages**

### B3: Background → Native Host
- `browser.runtime.sendNativeMessage("")` — JSON envelope with `{name: "core", data: ...}`
- **Confirmed message types:**
  - `{type: "Biometry", data: {type: "save", data: {secrets: [{accountUuid, userUuid, muk: {kty, kid, alg, k, ...}, srpX}]}}}` — **sends MUK (the master unlock key) and SRP-X to native app for biometric storage**
  - `{type: "Biometry", data: {type: "unlock", data: {accounts, useBiometry, useAppleWatch, ...}}}` — retrieves MUK + SRP-X from secure enclave
  - `{type: "Biometry", data: {type: "remove", data: {accounts, ...}}}` — removes stored secrets
  - `{type: "Biometry", data: {type: "biometryAvailability"}}` — checks if Touch ID / biometry is available
- 10-second timeout on all native messages
- dSecret proxy also flows through native messaging for MFA bypass
- **Risk: the MUK (symmetric key that unlocks everything) is serialized as JWK and sent over the native messaging channel**

### B4: Background → Remote Services
- HTTPS REST + WebSocket to 1Password infrastructure
- Snowplow telemetry to analytics endpoints
- Sentry error reporting
- DNS-over-HTTPS for Watchtower breach checking
- Partner APIs: Privacy.com, Fastmail, Brex, Kolide/Trelica

### B5: Background → Inline UI Frames
- `chrome.tabs.sendMessage` to push state to inline menus/modals
- **Risk: `relay-message-to-frames` and `targeted-message-to-inline-menu`** — background acts as a message relay between frames. If a compromised frame can influence the relay target, it could potentially spoof messages to other frames.

### B6: JS ↔ WASM (NOT a privilege boundary)
- Core crypto, trust log, trust verification, HPKE, mycelium all in WASM
- JS calls `rA.*` methods (80+ confirmed) which route into `op_wasm_b5x_bg`
- **Key material flows freely across this boundary:**
  - MUK exported from account handler via `.exportJwk()` → full JWK in JS heap
  - SRP-X cached on `CTX.session.auth.srpX` in JS
  - Decrypted passwords/OTPs returned from `rA.fillItem` / `rA.fieldValueByIdentifier` to JS
  - Save objects constructed via `rA.createSaveObject` — encrypted in WASM, ciphertext returned to JS
- WASM linear memory is directly accessible from same-origin JS
- **This is NOT a security boundary** — it's a code-sharing mechanism. A compromised background page can extract all key material.

## Sensitive Data Classes

| Class | Where Created | Where Stored | Where Transits | Notes |
|-------|--------------|-------------|----------------|-------|
| Master password | User input | **Timeboxed JS reference (5 min)**, then cleared | JS → WASM for key derivation, also set on `CTX.user.password` temporarily during sign-in then set to `undefined` | `setTimeout` clears after 5 min (`gkA = 5 * 60 * 1000`) |
| Master Unlock Key (MUK) | WASM (PBKDF2/HKDF from password + secret key) | **JS heap** as exportable JWK on account handler (`accountHandler.masterKey`) | JS → native messaging (biometry save), JS → WASM (re-auth) | **The crown jewel.** Can be exported via `.exportJwk()`. Sent to native app for biometric storage. |
| Secret Key (A3-XXXXX-...) | User input / stored in DB | Account handler, database | JS → WASM for SRP, exportable via `.exportSensitiveReadableString()` | Combined with password for key derivation |
| SRP-X | WASM (derived from MUK + Secret Key) | **JS heap** cached on `CTX.session.auth.srpX` | JS → native messaging (biometry save), JS → WASM (re-auth) | Used for re-authentication without master password |
| dSecret | Server / device | Account handler, native app | JS ↔ native messaging, JS → server for MFA | Device secret for MFA bypass on trusted devices |
| Session context (CTX) | WASM (`SA.initialize`) | **JS heap** on client object (`_dangerousInnerCTX`) | JS ↔ WASM, exportable via `SA.getInitializationExport()` | Contains session keys, account state. Note the `_dangerous` prefix — they know. |
| Vault/item encryption keys | WASM (decrypted from server keysets) | WASM memory (likely) | WASM internal | AES-GCM, AES-CBC. Exposed via `rA.fillItem` etc. |
| Item secrets (passwords, OTPs) | WASM (decrypted) | **JS heap** during fill | Background JS → `chrome.tabs.sendMessage` → content script → DOM | Plaintext in JS for the duration of fill |
| Passkey assertions | WASM | **JS heap** | Background → content script → page world postMessage | **Full chain traversal** through all trust zones |
| Credit card numbers | WASM (decrypted) | **JS heap** during fill | Same as item secrets | |
| Save objects | Content script (DOM capture) | Background | Content script → background → `rA.createSaveObject` (encrypted in WASM) → server | Encrypted with public key before storage |
| Biometry secrets bundle | JS (assembled from MUK + SRP-X) | Native app secure enclave | JS → `browser.runtime.sendNativeMessage` → OS keychain | `{muk: {kty, kid, alg, k, ext, key_ops}, srpX}` per account |
| Telemetry data | All contexts | Background (batched) | Background → Snowplow/Sentry | URLs, form hints, error stacks, account metadata |
| Feature flags | Remote server (Unleash) | Background (cached) | Background → all contexts | Controls security-relevant behavior |

See [key-hierarchy.md](key-hierarchy.md) for the full key derivation model.

## Network Endpoints (from CSP connect-src)

### 1Password Infrastructure
- `*.1password.com`, `*.1password.ca`, `*.1password.eu` (production)
- `wss://b5n.1password.com/ca/eu` (WebSocket notifications)
- `wss://b5n.ent.1password.com` (enterprise WebSocket)
- `*.b5dev.*`, `*.b5test.*`, `*.b5local.*`, `*.b5staging.*`, `*.b5rev.*` (dev/test/staging)
- `*.agilebits.com`
- `f.1passwordusercontent.com/ca/eu`, `a.1passwordusercontent.com/ca/eu` (file/asset CDN)

### Telemetry
- `com-1password-prod1.mini.snplow.net/com.snowplowanalytics.snowplow/tp2` (Snowplow)
- `telemetry.1passwordservices.com/com.snowplowanalytics.snowplow/tp2` (Snowplow alt)
- `b5x-sentry.1passwordservices.com` (Sentry)

### Partner Services
- `api.privacy.com`, `sandbox.privacy.com` (Privacy.com virtual cards)
- `www.fastmail.com`, `jmap.fastmail.com`, `betajmap.fastmail.com`, `api.fastmail.com` (email alias)
- `accounts.brex.com`, `platform.brexapis.com`, `*.staging.brexapps.com` (Brex cards)
- `app.kolide.com/ca/eu`, `api.kolide.com/ca/eu`, `auth.kolide.com/ca/eu` (Kolide EPM)
- `*.trelica.com` (Trelica app management)

### DNS-over-HTTPS (for Watchtower)
- `cloudflare-dns.com/dns-query`
- `private.canadianshield.cira.ca/dns-query`
- `9.9.9.10/dns-query` (Quad9)
- `unfiltered.joindns4.eu/dns-query`

### Local / Native
- `http://127.0.0.1:12519`, `:40978`, `:52115`, `:22287`, `:60685`, `:22322` (6 localhost ports)
- Likely redundant ports for the native helper broker / desktop app bridge (multiple ports for reliability across OS configurations)
- Native messaging also used via `browser.runtime.sendNativeMessage("")` for biometry, dSecret proxy, device trust

### Other
- `api.pwnedpasswords.com` (Have I Been Pwned API for Watchtower)
- `cache.agilebits.com` (icon/asset cache)

## Declarative Net Request Rules

`rules_1.json` defines one rule that modifies headers on DNS-over-HTTPS requests:
- **Removes** `User-Agent` header
- **Removes** `Accept-Language` header
- **Sets** `Origin` to `"null"`
- Applies to: Mullvad DNS, Cloudflare DNS, CIRA Canadian Shield, Quad9, joindns4.eu
- Resource type: `xmlhttprequest` only

This is a privacy-enhancing measure to prevent DNS providers from fingerprinting 1Password users when checking breached domains via Watchtower.

## Attack Surface Summary

### Critical
1. **MUK in JS heap** — the Master Unlock Key is stored as an exportable JWK in JS memory and sent over native messaging. A background page compromise (malicious update, browser bug, XSS in extension pages) exposes the key that decrypts everything.
2. **MUK + SRP-X sent to native app** — biometry save transmits `{muk: {k: "base64url_symmetric_key"}, srpX}` over native messaging JSON. If the native messaging channel is compromised, full account takeover is possible.
3. **Master password timeboxing only** — the password reference is cleared after 5 minutes via `setTimeout`, but the MUK derived from it persists in JS for the entire unlocked session. No explicit memory zeroing (JS GC handles it, which is non-deterministic).

### High Priority
4. **WebAuthn page-world IPC** — unauthenticated postMessage protocol, any page can participate
5. **Frame relay** (`relay-message-to-frames`) — background forwards messages between frames without clear origin validation (needs verification)
6. **Save object pipeline** — content script captures and transmits credentials (encrypted with public key)
7. **829 named messages** — massive handler surface in background, schema validation unknown
8. **`_dangerousInnerCTX`** — the session context object (containing session keys and auth state) is explicitly named "dangerous" by the developers, suggesting they recognize the risk of it being in JS

### Medium Priority
9. **External extension messaging** — `onMessageExternal` accepts messages from any Firefox extension
10. **6 localhost ports** — native bridge endpoints, protocol details unclear beyond biometry
11. **Web-accessible resources** — inline UI HTML files can be loaded by any page (fingerprinting, UI redressing)
12. **Re-authentication with cached MUK** — when a session expires (401), the extension re-authenticates using the cached MUK and SRP-X without user interaction (`executeWithReauth`). This means a stolen MUK enables silent re-auth.
13. **Delegated sessions** — background can request delegated sessions for re-auth, transferring session state between contexts

### Lower Priority
14. **Telemetry data classification** — what exactly is sent to Snowplow/Sentry
15. **Partner integration data minimization** — what's shared with Privacy.com, Fastmail, Brex, Kolide, Trelica
16. **`<all_urls>` + `webRequestBlocking`** — can observe/modify all web traffic
17. **Duo MFA tab injection** — extension opens a Duo MFA tab via `chrome.tabs.create`, monitors its URL for `duo_code` parameter, then closes it. The URL monitoring is done via `chrome.webRequest.onHeadersReceived`.
