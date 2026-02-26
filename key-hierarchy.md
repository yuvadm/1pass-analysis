# Key Hierarchy & Authentication Model

Reconstructed from static analysis of background.js sign-in, re-auth, biometry, and MFA flows.

## Key Derivation

```
Master Password (user input)
  + Secret Key (A3-XXXXX-XXXXX-XXXXX-XXXXX-XXXXX-XXXXX)
  │
  ├─ PBKDF2/HKDF ──→ Master Unlock Key (MUK)
  │                     │
  │                     ├─ Used to derive SRP-X for authentication
  │                     ├─ Exportable as JWK: {kty, kid, alg, k, ext, key_ops}
  │                     ├─ Stored in JS heap on account handler for entire unlocked session
  │                     └─ Optionally saved to OS secure enclave via native messaging (biometry)
  │
  └─ SRP protocol ──→ SRP-X (exchange value)
                        ├─ Cached in JS on CTX.session.auth.srpX
                        ├─ Used for re-authentication without password
                        └─ Optionally saved to OS secure enclave via native messaging (biometry)
```

## Key Storage Locations

| Key | JS Heap | WASM Memory | IndexedDB | Native/Secure Enclave | Server |
|-----|---------|-------------|-----------|----------------------|--------|
| Master Password | Timeboxed (5 min), then `undefined` | During derivation | Never | Never | Never |
| Secret Key | On account handler, in DB | During SRP | Encrypted in DB | Never | Never (derived server-side during SRP) |
| MUK | **Yes** — exportable JWK on account handler | Used for crypto ops | Never directly | **Yes** — via biometry save | Never |
| SRP-X | **Yes** — on `CTX.session.auth` | During SRP | Never | **Yes** — via biometry save | Never (used in SRP exchange) |
| dSecret | On account object | During MFA | In account DB | **Yes** — for dSecret proxy | Server-issued |
| Session keys | Inside CTX (JS) | Used for API calls | Never | Never | Session-scoped |
| Vault keys | Likely WASM-only | **Yes** — decrypted from server keysets | Never in plaintext | Never | Encrypted with keyset hierarchy |
| Item field values | **Yes** — during fill | During decrypt | Never in plaintext | Never | Encrypted in vault |

## Authentication Flows

### Primary Sign-In (password-based)

```
User enters: email + master password + secret key
  → SecretKey.fromInput() validates format
  → SA.signIn(ctx, credentials, options)
    → WASM: PBKDF2/HKDF derives MUK from password + secret key
    → WASM: SRP exchange with server
    → Server validates SRP proof
    → If domain_changed: retry with new host (max 1 redirect)
    → If mfa_required: enter MFA flow (see below)
    → CTX populated with session, account, user
  → Account overview fetched (getAccountWithAttrs)
  → Password set on CTX.user.password temporarily, then set to undefined
  → Password timeboxed in closure (5 min) for Duo MFA re-use
```

### MUK-based Re-Authentication (silent, no user interaction)

When a server request returns 401 (authentication required), the extension re-authenticates using cached credentials:

```
executeWithReauth(operationName, callback)
  → Try operation
  → If 401 error:
    1. Try delegated session (from another signed-in context)
    2. If that fails, try MUK + SRP-X re-auth:
       → buildMukSrpXCredentials():
         - Export MUK as JWK from account handler
         - Export Secret Key as readable string
         - Package with email, UUID, srpX, dSecret
       → Initialize new CTX with these credentials
       → SRP exchange with server using MUK (no password needed)
    3. If new context requires auth migration: sign out
    4. Retry original operation
```

**Security implication:** As long as the MUK is in memory (entire unlocked session), the extension can re-authenticate silently. No user interaction required.

### Biometric Unlock

```
Save flow (when unlocking with password succeeds):
  → fkA(): for each unlocked account handler:
    → Export MUK as JWK
    → Get SRP-X from client context
    → Package: {accountUuid, userUuid, muk: {kty, kid, alg, k, ...}, srpX}
  → browser.runtime.sendNativeMessage("", {
      name: "core",
      data: JSON.stringify({type: "Biometry", data: {type: "save", data: {secrets: [...]}}})
    })
  → Native app stores in OS secure enclave (Touch ID / Watch)

Unlock flow (biometric prompt):
  → browser.runtime.sendNativeMessage("", {
      name: "core",
      data: JSON.stringify({type: "Biometry", data: {type: "unlock", data: {accounts: [...]}}})
    })
  → Native app retrieves from secure enclave after biometric verification
  → Returns: {secrets: [{accountUuid, muk: JWK, srpX}], userFallback, userCancel}
  → For each returned secret:
    → Import MUK JWK back into crypto key
    → Sign in using MUK + SRP-X (same as re-auth flow)
  → On enrollment change (e.g., new fingerprint): auto-disable and re-enroll
```

### Duo MFA

```
If server requires MFA and Duo is enabled:
  → Password timeboxed in closure (5 min reuse window)
  → CTX.user.password set to undefined (cleared from context)
  → ZR class (Duo authenticator):
    → chrome.tabs.create(authURL) — opens Duo tab
    → chrome.webRequest.onHeadersReceived — monitors for redirect to duo-sign-in
    → chrome.tabs.onUpdated — watches for duo_code URL parameter
    → On code received: tabs.remove(duoTab), resolve promise
    → 5-minute timeout on entire flow
  → SA.completeMFASignInWithDuo(ctx, {type: "duov4", code}, options)
  → If password still in timebox: restore to CTX.user.password for session completion
  → Password reference cleared again after sign-in completes
```

### dSecret MFA Bypass

```
If server requires MFA and dSecret is available:
  → SA.completeMFASignInWithDSecret(ctx, dSecret, options)
  → If dSecret is invalid (device deauthorized): fall through to Duo
  → On success: mark account as not needing offline MFA

dSecret Proxy (via native app):
  → _QA(accountIdentifier, sessionId)
  → If desktop app connected: desktopConnectionManager.requestDsecretProxy(...)
  → Native app provides dsecretHmac + deviceUUID
  → SA.completeMFASignInWithDSecretProxy(ctx, dsecretHmac, deviceUUID, options)
```

### Delegated Sessions

```
When one client context needs to re-auth:
  → requestDelegatedSession(accountUUID)
  → Returns: {notifier, sessionKey}
  → Export initialization state from old CTX
  → Build new CTX with delegated sessionKey + old init state
  → If transport token enabled: transfer from old CTX
```

## Password Handling Details

### Timebox Mechanism
```javascript
var gkA = 5 * 60 * 1000;  // 5 minutes
function _7e(password) {
    let j = { password };
    setTimeout(() => { j.password = undefined }, gkA);
    return (checkOnly) => {
        let t = j.password;
        return checkOnly && t ? true : (j.password = undefined, t);
    };
}
```

The password is held in a closure. After 5 minutes, it's set to `undefined`. The getter function either checks existence (without consuming) or reads and clears. This is used for Duo MFA re-use — if the user needs to complete MFA, the password must still be available.

### Password on CTX
During sign-in: `CTX.user.password = accountPassword`
After account load: `CTX.user.password = undefined`
During Duo flow: password temporarily removed from CTX, restored from timebox after Duo completes, then removed again.

## Crypto Algorithms (from code)

| Algorithm | Usage |
|-----------|-------|
| PBKDF2 | Master password → key derivation |
| HKDF | Key derivation (with SHA-256) |
| AES-GCM | Vault item encryption/decryption |
| AES-CBC | Legacy vault item encryption |
| RSA-OAEP | Key wrapping, keyset encryption |
| ECDSA | Signing (trust log, device trust) |
| HMAC (SHA-256) | dSecret, message authentication |
| SHA-256 | General hashing |
| SHA-1 | HIBP password prefix hashing |
| SRP | Authentication protocol (custom `SrpMethod`, `SrpStarter`, `SrpV`, `SrpX`) |
| NaCl | `lessSafeOpenNaCl` — likely for legacy or specific crypto operations |
| HPKE (RFC 9180) | Item sharing, sealed encryption |
| X25519 | Key agreement (`JwkX25519PriKey`, `JwkX25519PubKey`) |
| Noise-like protocol | Mycelium relay handshake |

## Security Observations

1. **No memory zeroing in JS.** MUK, SRP-X, decrypted passwords — all rely on JS garbage collection. The `undefined` assignment removes the reference but doesn't zero the underlying memory. GC timing is non-deterministic.

2. **MUK is the skeleton key.** With the MUK + SRP-X, anyone can silently authenticate as the user and access all vaults. These values persist in JS for the entire unlocked session and are optionally stored in the OS secure enclave.

3. **The `_dangerousInnerCTX` naming** is honest — the developers know the CTX object in JS is sensitive. It contains everything needed to make authenticated requests.

4. **Re-auth is transparent to the user.** If a session expires, the extension silently re-authenticates using cached MUK + Secret Key + SRP-X. Users may not realize their session was re-established.

5. **Duo MFA password reuse window** — the master password is kept for up to 5 minutes in a closure specifically to survive the Duo MFA flow. If the Duo flow takes less than 5 minutes, the password is available for that window.

6. **Biometry stores MUK + SRP-X in the clear (as JWK)** in the OS secure enclave. The native messaging channel carries this data as plaintext JSON. If the native messaging protocol is intercepted (unlikely but possible on compromised systems), the MUK is exposed.
