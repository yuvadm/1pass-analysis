# Message Catalog

All message handlers extracted from static analysis. Messages flow via `chrome.runtime.sendMessage` / `chrome.runtime.onMessage`.

## Background Message Router

The main handler registration (`m5({...})`) in `background.js` registers these content-script → background handlers:

### Core / Account
| Message | Notes |
|---------|-------|
| `get-account-list` | Also exported as `b5xHandlers` for b5 web app |
| `get-default-account-info` | |
| `open-extension-welcome-page-for-onboarding` | |

### b5 Web App Integration
| Message | Notes |
|---------|-------|
| `open-swe-native-wrapper` | Open native app wrapper from SWE |
| `sign-in-succeeded` | b5 web app sign-in completed |
| `b5-session-init` | Initialize session from b5 web app |
| `b5x-request-session-init` | Extension requests session from b5 |
| `b5-sso-complete` | SSO flow completed on b5 |
| `b5x-credential-hydration` | Hydrate credentials from b5 |

### Frame/Inline Management
| Message | Notes |
|---------|-------|
| `relay-message-to-frames` | **Background relays messages between frames** — attack surface |
| `targeted-message-to-inline-menu` | Send message to specific inline menu |
| `refresh-can-request-unlock` | |
| `get-frame-manager-configuration` | |

### Autofill / Inline Suggestions
| Message | Notes |
|---------|-------|
| `get-inline-suggestions` | Content script requests fill suggestions |
| `perform-inline-action` | User selected an inline action |
| `record-inline-menu-render-event-all-accounts` | Telemetry for inline menu render |

### Save Flow
| Message | Notes |
|---------|-------|
| `add-save-object` | Content script submits credential to save |
| `get-save-object-public-key` | Get public key for encrypting save objects |

### WebAuthn / Passkeys
| Message | Notes |
|---------|-------|
| `create-credential` | Passkey creation request |
| `get-credential` | Passkey authentication request |
| `should-intercept-webauthn-request` | Content script asks if it should intercept WebAuthn |

### Privacy.com Integration
| Message | Notes |
|---------|-------|
| `enable-privacy-integration` | |
| `get-privacy-enable-dialog-configuration` | |
| `get-privacy-create-dialog-configuration` | |
| `fill-new-privacy-card` | |

### Email Alias (Fastmail)
| Message | Notes |
|---------|-------|
| `get-email-alias-dialog-configuration` | |
| `start-email-alias-session` | |
| `generate-email-alias` | |
| `cancel-email-alias-session` | |
| `fill-new-email-alias` | |

### Brex Integration
| Message | Notes |
|---------|-------|
| `get-brex-dialog-configuration` | |
| `fill-new-brex-card` | |

### Shell Plugins (AI Agent Integration)
| Message | Notes |
|---------|-------|
| `shell-plugins-notification-config` | |
| `shell-plugins-dismiss-notifications` | |
| `shell-plugins-save-in-1password-notification` | Credential detected in AI browsing agent |
| `shell-plugins-fallback-notification` | |
| `shell-plugins-site-config` | |
| `shell-plugins-item-saving-prompt` | |
| `shell-plugins-get-last-detected-credentials` | |
| `shell-plugins-set-last-detected-credentials` | |

### Error/Logging
| Message | Notes |
|---------|-------|
| `report-error` | Structured error reporting from any context |
| `health-check-request` | (in health-check.js) Returns `health-check-response` with `"alive"` |

## Content Script → Background Messages (from inline scripts)

### inject-content-scripts.js
- `login-detection-is-enabled` — query feature flag
- `new-log-event` — log from popup context
- `new-tab-log-event` — log from content/extension page context
- `report-error` — structured error report

### injected.js (page managers)
Sends all of the `m5` handler messages listed above, plus internal frame management:
- `forward-active-field-details`
- `forward-inline-menu-position`
- `frame-takes-focus`
- `provide-frame-origin`
- `filter-inline-menu`
- `focus-inline-menu`
- `add-scroll-and-resize-event-listeners`
- `remove-scroll-and-resize-event-listeners`
- `edited-state-changed`

### b5.js (1Password web app pages)
- `b5-session-init`, `b5x-request-session-init`, `b5-sso-complete`
- `b5x-credential-hydration`
- `open-swe-native-wrapper`, `sign-in-succeeded`
- `open-extension-welcome-page-for-onboarding`

### heuristics.js (login detection)
- `LOGIN_EVENT`, `LOGIN_STEP` — login heuristic detection events

### modal.js (dialogs)
- `can-request-unlock` — check if unlock is requestable
- Privacy.com, email alias, Brex dialog messages (listed above)

### notification.js
- `DeviceTrust` — device trust notification
- `report-error`

### menu.js (inline menu)
- `get-inline-suggestions`, `perform-inline-action`
- `focus-page`, `save-item`, `request-verification-token`

## Background → Content Script Messages

Background uses `chrome.tabs.sendMessage` (7 occurrences) to push messages to tabs. Also uses `chrome.scripting.executeScript` (4 occurrences) for direct injection. Specific message names sent to tabs need deeper analysis of the minified code.

## Background Broadcast Events (internal)

Observed event names used in background's internal pub/sub system:
- `accounts-and-vaults-changed`
- `accounts-locked`
- `can-request-unlock-changed`
- `unleash-features-changed`
- `extension-first-survey`
- `unified-panel-update`

## Message Volume

Total unique `name:"..."` strings in background.js: **829** (includes API method names, crypto algorithm names, vault/group names, etc. — not all are message handler names).
