# 1Password Firefox Extension Security Review

Static analysis of the 1Password browser extension for Firefox (`1pass-xpi`), version **8.12.2.38** (stable channel, built 2026-02-11).

## Documentation

| File | Description |
|------|-------------|
| [architecture.md](architecture.md) | Runtime topology, boot pipeline, permission model, WASM core interface |
| [key-hierarchy.md](key-hierarchy.md) | Key derivation, MUK lifecycle, auth flows, biometry, Duo MFA, crypto algorithms |
| [message-catalog.md](message-catalog.md) | Complete message handler map extracted from source, including native messaging protocol |
| [webauthn-analysis.md](webauthn-analysis.md) | WebAuthn monkey-patching, page-world IPC protocol, attack surface |
| [trust-boundaries.md](trust-boundaries.md) | Trust zones, dataflow, sensitive data classes, attack surfaces |
| [telemetry.md](telemetry.md) | Snowplow analytics, Sentry error reporting, DNS privacy proxy |
| [TODO.md](TODO.md) | Planned future work and open questions |

## Source Material

- `1pass-xpi/` — Extracted extension package (XPI/ZIP), Mozilla-signed
- Extension ID: `{d634138d-c276-4fc8-924b-40a0ea21d284}`
- Manifest version: 2 (Firefox)
- Gecko minimum: 128.0
