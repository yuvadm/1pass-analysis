# Telemetry & Error Reporting

## Snowplow Analytics

The extension uses Snowplow for product analytics/telemetry.

### Endpoints
- `com-1password-prod1.mini.snplow.net/com.snowplowanalytics.snowplow/tp2`
- `telemetry.1passwordservices.com/com.snowplowanalytics.snowplow/tp2`

### Schema
Uses standard Snowplow protocol:
- `iglu:com.snowplowanalytics.snowplow/payload_data/jsonschema/1-0-4`
- `iglu:com.snowplowanalytics.snowplow/unstruct_event/jsonschema/1-0-0`
- `iglu:com.snowplowanalytics.snowplow/contexts/jsonschema/1-0-0`

### Context Data Attached
From code analysis, telemetry events include:
- `accountCreationDate` — when the account was created
- `accountTier` — subscription tier name
- `accountType` — account type
- `userRole` — array of role strings (e.g., "guest")
- `telemetryStatus` — whether user opted in
- `billingProvider` (if available)
- `idp` — identity provider (if SSO)
- `activeMembers` count (if available)

### Event Types Observed
- Fill telemetry (`b5x-filling-saving-telemetry` feature flag)
- USO (Universal Sign-On) telemetry (`reportUsoAction`, dedicated endpoint `/api/v1/uso/telemetry/{action}`)
- Inline menu render events
- Pre-auth telemetry (`pre-auth-telemetry-data-collection`)
- B2B telemetry (`b2b-telemetry-setting-UI`)
- Activation hub events
- Iterable quest tracking
- Performance traces (`sendPerformanceTraces`, `sendPreAuthPerformanceTraces`)

### Opt-Out Controls
- `telemetryOptOut` — policy flag (can be set by enterprise admin)
- `telemetry_modal_opt_in` / `telemetry_modal_opt_out` — user choice
- `essential_setup_telemetry_modal` — first-run consent modal
- `essential_setup_telemetry_modal_remind_me_later` — defer choice
- `Cj.isOptedInToTelemetry(user)` — per-user check before sending

### Redaction
- Fill telemetry errors explicitly use `"Fill telemetry error: <redacted>"` pattern
- `TelemetryString` type with `assertSafeForTelemetry` — separate type system from `LoggableString` to enforce what can be sent in telemetry vs what can be logged locally
- `LoggableString` type with `assertLogSafe` — controls what appears in local logs

### High-Volume Event Gating
- `telemetry-high-volume-events-denied` — mechanism to suppress high-frequency events

## Sentry Error Reporting

### Configuration
- DSN: `4f3669ef553b434e86845ecd15a92e28:acaac9ba43f34f61917ae843f639604c@b5x-sentry.1passwordservices.com/4505427098533888`
- Project ID: `4505427098533888`
- Sample rate: 0.5 (when conditions met) or 1.0
- App version format: `b5x@{version}-{channel}`

### What's Reported
- Uncaught exceptions and unhandled promise rejections (automatic)
- Explicit `report-error` messages from any extension context
- Stack traces with source file references (source maps available as web-accessible resources)
- DOM breadcrumbs (console, DOM events, XHR, fetch, history changes)
- Debug IDs embedded in every JS file for stack trace correlation

### Sentry Debug IDs
Every JS file starts with a Sentry debug ID registration:
```javascript
e._sentryDebugIds = e._sentryDebugIds || {};
e._sentryDebugIds[stackTrace] = "unique-debug-id";
```
This enables source map correlation for crash reports without serving unobfuscated source.

### Privacy Controls
- `devtools-send-local-sentry` — developer toggle for local Sentry testing
- `sendLocalSentryEnabled()` — gate check
- URL sanitization via `rp(url, maxLength)` before sending
- Exception value sanitization
- Request URL sanitization
- Standard Sentry filters: `Script error`, `ResizeObserver loop`, `googletag` excluded

### Sentry Integrations Active
From code analysis, these Sentry integrations are initialized:
- Console breadcrumbs
- DOM event breadcrumbs
- XHR breadcrumbs
- Fetch breadcrumbs
- History breadcrumbs
- Unhandled rejection capture
- Global error handler

## Logging Infrastructure

### LogReporter Class
Every extension context creates a `LogReporter` instance:
- Generates a session ID (random base-36)
- Records `performance.timeOrigin` for timing correlation
- Captures context type (`ContentScript`, `Popup`, `ExtensionPage`)
- Automatically captures uncaught exceptions and unhandled rejections

### Log Event Structure
```
{
  id: random_uint32_base36,
  runtime: { context, initializer, session, origin },
  message: [...],
  timestamp: performance.now(),
  fileName, lineNumber, severity, prefix, highlight
}
```

### Log Routing
| Context | Message Name | Destination |
|---------|-------------|-------------|
| Popup | `new-log-event` | Background |
| ContentScript | `new-tab-log-event` | Background |
| ExtensionPage | `new-tab-log-event` | Background |

### DevTools Panel
`devtools/panels.html` + `panels.js` (353KB) provides a "Logging" panel in browser DevTools for real-time log inspection. Created via `chrome.devtools.panels.create`.

## DNS-over-HTTPS Privacy Proxy

Watchtower breach-checking queries use DNS-over-HTTPS to check domain breach status. To prevent DNS providers from fingerprinting users:

### Header Modification (rules_1.json)
On requests to DoH providers:
- **Removes** `User-Agent` — prevents browser/OS fingerprinting
- **Removes** `Accept-Language` — prevents locale fingerprinting
- **Sets** `Origin: null` — prevents referrer-based correlation

### DoH Providers
| Provider | Endpoint |
|----------|----------|
| Mullvad DNS | `dns.mullvad.net` |
| Cloudflare DNS | `cloudflare-dns.com` |
| CIRA Canadian Shield | `private.canadianshield.cira.ca` |
| Quad9 | `9.9.9.10` |
| joindns4.eu | `unfiltered.joindns4.eu` |

Multiple providers likely used for redundancy and to avoid single-provider correlation.

### pwnedpasswords.com
Direct HTTPS API calls to `api.pwnedpasswords.com` for Have I Been Pwned password checking. Uses k-anonymity (prefix-based) API — only the first 5 hex characters of the SHA-1 hash are sent.
