# TODO — Future Work

## High Priority

### Deep Reverse Engineering of background.js
- [ ] Beautify/pretty-print the 2.9MB background.js for readable analysis
- [ ] Map all `m5({...})` handler implementations — trace what each handler does after receiving a message
- [ ] Determine if message schema validation exists at the background router level (critical for security posture)
- [ ] Identify the `sdj(wd)` call — appears to register additional handlers or middleware
- [ ] Trace the `VA()` function used for broadcasting events to tabs

### Native Messaging Protocol
- [ ] Identify the native app ID used with `chrome.runtime.sendNativeMessage`
- [ ] Map the command/response schema for native messaging
- [ ] Determine what operations are delegated to the native app vs handled in-extension
- [ ] Analyze the 6 localhost ports (12519, 40978, 52115, 22287, 60685, 22322) — what protocol, what data
- [ ] Check if localhost connections use any authentication/signing

### Frame Relay Security
- [ ] Trace `relay-message-to-frames` handler (`eij`) — does it validate sender frame? Does it restrict what messages can be relayed?
- [ ] Trace `targeted-message-to-inline-menu` handler (`tij`) — same questions
- [ ] Determine if a compromised iframe can spoof messages to another frame's inline menu
- [ ] Check origin validation on all frame-relay paths

### WebAuthn Protocol Hardening Assessment
- [ ] Test if a malicious page can inject fake `op-window-syn-ack` before the real extension responds
- [ ] Test the 100ms re-patching race condition — can a page reliably intercept credentials between overwrite and re-patch?
- [ ] Analyze what happens if two extensions both try to intercept WebAuthn
- [ ] Check if the `stopImmediatePropagation` ordering is reliable at `document_start`

### Save Object Pipeline
- [ ] Trace the save flow end-to-end: DOM capture → `add-save-object` → public key encryption → storage
- [ ] Verify the public key used for save object encryption is authenticated (not injectable)
- [ ] Check what happens if a page injects fake form data before the save prompt

## Medium Priority

### WASM Module Analysis
- [ ] Analyze exported functions from `op_wasm_b5x_bg` — what crypto primitives are exposed to JS?
- [ ] Determine if key material ever crosses the WASM→JS boundary in plaintext
- [ ] Check `confidential_computing_bg` — what confidential computing features are used and where?
- [ ] Analyze `b5_mycelium_bg` — is this the Mycelium relay for remote autofill? What's the protocol?
- [ ] Map HPKE usage — what is encrypted with HPKE vs AES-GCM vs RSA-OAEP?

### External Extension Messaging
- [ ] Trace all 3 `onMessageExternal` handler registrations
- [ ] Determine what messages are accepted from external extensions
- [ ] Check if there's any authentication/validation of the sender extension ID

### Large Chunk Analysis
- [ ] Analyze `chunk-OJR52IF5.js` (1.9MB) — likely contains the bulk of vault/account logic
- [ ] Analyze `chunk-MSIWLBOQ.js` (804KB) — likely UI component library or crypto support
- [ ] Analyze `chunk-22IBMJDR.js` (138KB)
- [ ] Analyze `chunk-QEIU26VY.js` (92KB)
- [ ] Analyze `chunk-PQH6ALAA.js` (156KB)

### Credential Fill Path
- [ ] Trace the fill flow: background resolves item → decrypts → sends to content script → injects into DOM
- [ ] Identify where decrypted password/OTP/card values exist in JS memory and for how long
- [ ] Check if clipboard operations (`clipboardWrite`) properly clear after timeout
- [ ] Analyze `fill-generated-password` flow — is the generated password ever in plaintext outside WASM?

### Partner Integration Data Minimization
- [ ] Privacy.com: what data is sent to `api.privacy.com`? Card params, user identity?
- [ ] Fastmail: what data is sent to Fastmail JMAP? Email addresses, domain info?
- [ ] Brex: what data flows to `platform.brexapis.com`?
- [ ] Kolide: what health/device data is shared? Analyze `kolide.js` (60KB)
- [ ] Trelica: what app catalog data is exchanged?

### Secure Remote Autofill (director.ai)
- [ ] Analyze `secure-remote-autofill-start-pairing.js` (57KB) — pairing protocol
- [ ] Analyze `secure-remote-autofill-complete-pairing.js` (59KB) — completion flow
- [ ] Determine what crypto is used for the remote autofill channel (likely Mycelium + HPKE)
- [ ] Check if the pairing is bound to specific devices/sessions

### Shell Plugins / AI Agent Integration
- [ ] Map the full shell-plugins subsystem — what AI browsing agents are supported?
- [ ] Analyze credential detection in agent contexts (Browserbase, BrowserUse icons present in chunks)
- [ ] Check what prompts/confirmations exist before saving agent-detected credentials
- [ ] Assess risk of AI agents leaking credential data through their own telemetry

## Lower Priority

### Telemetry Deep Dive
- [ ] Extract all Snowplow event schemas (iglu schema references)
- [ ] Determine exactly what URL/page data appears in telemetry events
- [ ] Verify redaction is applied consistently (not just in fill telemetry)
- [ ] Check if error stack traces sent to Sentry contain sensitive URL parameters

### Popup / App UI
- [ ] Analyze `popup/index.js` (399KB) — what privileged operations can be triggered from popup?
- [ ] Analyze `app/app.js` (682KB) — full window UI capabilities
- [ ] Check for any XSS vectors in UI rendering of vault item data

### Login Detection Heuristics
- [ ] Analyze `heuristics.js` — what classifies as a login page?
- [ ] Check for false positive scenarios that could trigger unintended autofill
- [ ] Verify `LOGIN_EVENT` and `LOGIN_STEP` don't leak sensitive page content

### Internationalization
- [ ] Check if locale-specific code paths have different security properties
- [ ] Analyze `assets/js/messages.i18n-*.js` files for any embedded data beyond translations

### Web-Accessible Resource Fingerprinting
- [ ] Document exactly which resources are web-accessible and can be probed by any site
- [ ] Assess extension detection/fingerprinting risk from these resources
- [ ] Check if `*.js.map` being web-accessible leaks useful information to attackers

## Tooling & Infrastructure

### Analysis Setup
- [ ] Set up JS beautifier pipeline (prettier/esbuild) for all minified files
- [ ] Build automated string extraction for message names, URLs, feature flags
- [ ] Set up AST-based analysis for tracing message handler call chains
- [ ] Create a dynamic analysis harness (load extension in test Firefox profile, intercept messages)

### Documentation
- [ ] Create a visual architecture diagram (Mermaid/D2)
- [ ] Build a cross-reference index: message name → handler file → handler function → effects
- [ ] Document all feature flags and their security implications
