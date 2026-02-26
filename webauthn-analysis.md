# WebAuthn / Passkey Mediation Analysis

The extension intercepts all WebAuthn (passkey) operations on every HTTPS page. This is the most security-sensitive component because it runs code in the **page world** (not the extension's isolated world).

## Injection Chain

```
manifest.json
  └─ content_scripts[1]: webauthn.js + webauthn-listeners.js
       matches: https://*/* and http://localhost/*
       run_at: document_start
       all_frames: true

webauthn.js (content script, ISOLATED world)
  └─ Contains full inline menu / frame manager code (93KB)
  └─ Communicates with background via chrome.runtime.sendMessage

webauthn-injection-helper.js (bundled in webauthn.js)
  └─ Creates a <script> tag pointing to webauthn-listeners.js
  └─ Injects into document.documentElement at document_start
  └─ Removes the <script> element after injection
  └─ Guards: skips injection on Dropbox download pages

webauthn-listeners.js (runs in PAGE WORLD — untrusted)
  └─ Monkey-patches navigator.credentials.create/get
  └─ Monkey-patches PublicKeyCredential static methods
  └─ Communicates via window.postMessage IPC protocol
```

## The Page-World Monkey-Patch

`webauthn-listeners.js` replaces these browser APIs globally:

| Original API | Replacement | Behavior |
|-------------|-------------|----------|
| `navigator.credentials.create()` | `N()` | Routes through extension if intercept enabled, else falls back to browser |
| `navigator.credentials.get()` | `H()` | Routes through extension if intercept enabled, else falls back to browser |
| `PublicKeyCredential.isConditionalMediationAvailable()` | `W()` | Returns `true` if extension intercepts |
| `PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable()` | `K()` | Returns `true` if extension intercepts |
| `PublicKeyCredential.getClientCapabilities()` | `U()` | Returns hardcoded capability set if extension intercepts |

### Firefox-Specific Injection

Uses Firefox's privileged APIs when available:
- `exportFunction()` — safely export JS functions across compartments (Xray wrappers)
- `cloneInto()` — clone objects across compartments with `{cloneFunctions: true, wrapReflectors: true}`
- `window.wrappedJSObject` — access the page's actual JS global

If these aren't available (shouldn't happen on Firefox), falls back to direct property assignment.

### Re-Patching Loop

```javascript
setInterval(Y, 100)  // every 100ms
```

Function `Y()` checks if `navigator.credentials.create.length > 0` and `navigator.credentials.get.length > 0`. If not (indicating the patch was overwritten), and the same check fails on `wrappedJSObject`, it re-applies the patches. This is defensive against other extensions or page scripts that might also try to hook WebAuthn.

## Custom IPC Protocol (window.postMessage)

Since `webauthn-listeners.js` runs in the page world, it cannot use `chrome.runtime.sendMessage`. Instead, it uses a custom protocol over `window.postMessage`:

### Message Types
| Type | Direction | Purpose |
|------|-----------|---------|
| `op-window-syn` | Page → Extension | Initial handshake request |
| `op-window-syn-ack` | Extension → Page | Handshake acknowledgment |
| `op-window-direct-request` | Page → Extension | Actual request payload |
| `op-window-direct-response` | Extension → Page | Response payload |
| `op-window-abort` | Page → Extension | Cancel in-progress request |

### Protocol Flow
1. Page-world code generates a random `source` ID and `msgId`
2. Sends `op-window-syn` with a route name (e.g., `"get-credential"`)
3. Extension content script (in isolated world) receives via `window.addEventListener("message")`
4. If it has a handler for that route, responds with `op-window-syn-ack` including its own `source` ID
5. Page sends `op-window-direct-request` targeted to the ack'd source
6. Extension processes request (via `chrome.runtime.sendMessage` to background)
7. Extension sends `op-window-direct-response` back

### Message Validation
Each message must have:
- `msgId` (non-empty string)
- `source` (non-empty string)
- `name` (non-empty string)
- `type` (one of the 5 valid types)

Messages are ignored if `source` matches self (prevents echo). Targeted messages (`destination`) are only processed if they match the receiver's source ID.

### Registered Routes

Page world → Extension:
- `get-credential` — passkey authentication
- `create-credential` — passkey registration
- `should-intercept-webauthn` — check if extension should handle WebAuthn

Extension → Page world:
- `update-settings` — push `authenticatePasskeys` boolean to page

## Intercept Decision Flow

```
Page calls navigator.credentials.get(options)
  → f() checks: should we intercept?
    → If R (authenticatePasskeys) is cached, use it
    → If not cached, send "should-intercept-webauthn" to extension
    → If response is true: route through extension (V/L functions)
    → If response is false/error: call original browser API
```

## Credential Serialization

WebAuthn credentials are serialized across the postMessage boundary:
- `ArrayBuffer` / typed arrays → regular `Array` (via `O()` function)
- Binary fields Base64URL-encoded for JSON transport
- `AuthenticatorAssertionResponse` and `AuthenticatorAttestationResponse` are reconstructed with proper prototypes on return

Response reconstruction explicitly sets `Object.setPrototypeOf(result, PublicKeyCredential.prototype)` so `instanceof` checks pass on the page side.

## Error Handling and Fallbacks

| Scenario | Behavior |
|----------|----------|
| Extension doesn't respond to SYN within 1000ms | Falls back to browser WebAuthn |
| Extension returns handler error | Falls back to browser WebAuthn |
| Extension returns `"timeout"` | Throws `NotAllowedError` DOMException |
| Extension returns `"user-cancelled"` (conditional mediation) | Returns never-resolving promise |
| Extension returns `"user-cancelled"` (non-conditional) | Returns `null` |
| Extension returns `"fallback-requested"` (conditional) | Calls browser API without `mediation` option |
| Extension returns `"fallback-requested"` (non-conditional) | Falls back to browser WebAuthn |
| Extension returns `"duplicate"` (create) | Throws `InvalidStateError` DOMException |
| AbortSignal fires | Sends `op-window-abort`, throws `AbortError` DOMException |

## Dropbox Exclusion

The injection helper explicitly skips WebAuthn patching on Dropbox download pages:
```javascript
window.location.hostname.includes("dropbox") &&
  (window.location.pathname.includes("get") ||
   window.location.search.includes("download_id"))
```
This suggests a known compatibility issue with Dropbox's download flow.

## Security Considerations

### Attack Surface
1. **Any page can send `op-window-syn` messages** — the protocol is unauthenticated at the transport layer. Route name validation and the SYN/ACK handshake provide some protection, but a malicious page could race legitimate requests.

2. **The `source` IDs are random 32-bit integers** (base-36 encoded). With ~4 billion possible values, a brute-force collision is impractical per-request, but the entropy is modest.

3. **Re-patching via `setInterval`** means a page that continuously overwrites `navigator.credentials` creates a race condition window between overwrites and re-patches.

4. **`stopImmediatePropagation`** is called on matched messages, preventing other listeners from seeing them. But a page could register its listener before the extension's (it runs at `document_start`, but timing isn't guaranteed in all edge cases).

5. **Conditional mediation get requests** have their `timeout` stripped (`publicKey.timeout = undefined`), which means they can hang indefinitely until the extension responds.

6. **The `signal` property is stripped** from options before sending to the extension. The extension's abort handling is through its own `op-window-abort` message type.

### Hardcoded Capability Advertisement
When the extension intercepts, `getClientCapabilities()` returns a hardcoded set claiming support for `conditionalGet`, `hybridTransport`, `passkeyPlatformAuthenticator`, `prf` extension, etc. — regardless of actual platform capabilities.
