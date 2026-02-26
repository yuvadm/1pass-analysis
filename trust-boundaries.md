# Trust Boundaries & Dataflow

## Trust Zones

### Zone A — Background (highest extension privilege)
- Full access to all `chrome.*` APIs
- Holds vault state, account sessions, crypto keys (in WASM memory)
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
- `chrome.runtime.sendNativeMessage` (2 occurrences in background.js)
- Native app connection initialized during background startup (`initializeNativeAppConnection`)
- Trust level: equivalent to or higher than extension (system-level process)

### Zone E — Remote Services (1Password cloud)
- Extensive `connect-src` allowlist (see below)
- WebSocket connections for real-time sync (`wss://b5n.*`)
- REST APIs for account management, vault operations

### Zone F — WASM Modules (crypto boundary)
- 7 WASM modules loaded with `wasm-unsafe-eval`
- Crypto operations are isolated inside WASM linear memory
- JS ↔ WASM boundary is a trust boundary: JS passes data in, WASM performs crypto, returns results
- Key material should ideally never exist in JS heap (only WASM memory)

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
- `chrome.runtime.sendNativeMessage` — structured JSON messages
- Mycelium protocol (WASM) may also use this channel for relay
- **Needs analysis: what commands can be sent? What data flows back?**

### B4: Background → Remote Services
- HTTPS REST + WebSocket to 1Password infrastructure
- Snowplow telemetry to analytics endpoints
- Sentry error reporting
- DNS-over-HTTPS for Watchtower breach checking
- Partner APIs: Privacy.com, Fastmail, Brex, Kolide/Trelica

### B5: Background → Inline UI Frames
- `chrome.tabs.sendMessage` to push state to inline menus/modals
- **Risk: `relay-message-to-frames` and `targeted-message-to-inline-menu`** — background acts as a message relay between frames. If a compromised frame can influence the relay target, it could potentially spoof messages to other frames.

### B6: JS → WASM
- Core crypto, trust log, trust verification, HPKE, mycelium all in WASM
- JS serializes data, calls WASM exports, receives results
- **Risk: incorrect serialization could leak or corrupt key material at the boundary**

## Sensitive Data Classes

| Class | Where Created | Where Used | Notes |
|-------|--------------|------------|-------|
| Master password / derived key | User input → WASM | WASM memory only (ideally) | PBKDF2/HKDF derivation |
| SRP verifier/session | WASM | Background ↔ Remote | Authentication protocol |
| Vault encryption keys | WASM (decrypted from server) | WASM memory | AES-GCM, AES-CBC |
| Item secrets (passwords, OTPs) | WASM (decrypted) | Passed to content script for fill | **Transits JS heap** |
| Passkey private keys | WASM | WASM → background → content → page world | **Full chain traversal** |
| Credit card numbers | WASM (decrypted) | Content script fill | |
| Session tokens | Background | Background ↔ Remote | |
| Device keys | Background/Native | Background | |
| Save objects (captured credentials) | Content script | Background → WASM → Remote | Encrypted with public key before transit |
| Telemetry data | All contexts | Background → Snowplow/Sentry | URLs, form hints, error stacks |
| Feature flags | Remote → Background | All contexts | Controls behavior |

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
- These are likely the native helper broker / desktop app bridge endpoints

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

### High Priority
1. **WebAuthn page-world IPC** — unauthenticated postMessage protocol, any page can participate
2. **Frame relay** (`relay-message-to-frames`) — background forwards messages between frames without clear origin validation (needs verification)
3. **Save object pipeline** — content script captures and transmits credentials (encrypted with public key)
4. **829 named messages** — massive handler surface in background, schema validation unknown

### Medium Priority
5. **External extension messaging** — `onMessageExternal` accepts messages from any Firefox extension
6. **6 localhost ports** — native bridge endpoints, protocol unknown
7. **Web-accessible resources** — inline UI HTML files can be loaded by any page (fingerprinting, UI redressing)
8. **WASM ↔ JS boundary** — key material handling across the boundary

### Lower Priority
9. **Telemetry data classification** — what exactly is sent to Snowplow/Sentry
10. **Partner integration data minimization** — what's shared with Privacy.com, Fastmail, Brex, Kolide, Trelica
11. **`<all_urls>` + `webRequestBlocking`** — can observe/modify all web traffic
