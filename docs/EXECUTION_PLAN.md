# Kipta – SMS Relay · Execution Plan

> Status: **Planning** · Platform: **Android (native Kotlin)** · Owner: Jobaer
> This document is the single source of truth for *what* we build, *how*, *in what order*, and *how we know it is done*. No code exists yet.

---

## Table of contents

1. [Product definition](#1-product-definition)
2. [Platform facts and constraints](#2-platform-facts-and-constraints)
3. [Architecture](#3-architecture)
4. [Module layout](#4-module-layout)
5. [Tech stack](#5-tech-stack)
6. [Data model](#6-data-model)
7. [The pipeline: Read → Filter → Match → Understand → Format → Send](#7-the-pipeline)
8. [Kill switches](#8-kill-switches)
9. [Network layer and the network console](#9-network-layer-and-the-network-console)
10. [Security and privacy design](#10-security-and-privacy-design)
11. [Threat model](#11-threat-model)
12. [Webhook contract](#12-webhook-contract)
13. [UX and design system (VPN-style)](#13-ux-and-design-system-vpn-style)
14. [Screen-by-screen specification](#14-screen-by-screen-specification)
15. [Edge cases and platform quirks](#15-edge-cases-and-platform-quirks)
16. [Testing strategy](#16-testing-strategy)
17. [CI/CD and repository hygiene](#17-cicd-and-repository-hygiene)
18. [Distribution and Play policy](#18-distribution-and-play-policy)
19. [Phased delivery plan with tasks and acceptance criteria](#19-phased-delivery-plan)
20. [Timeline summary](#20-timeline-summary)
21. [Risks and mitigations](#21-risks-and-mitigations)
22. [Open questions](#22-open-questions)
23. [Definition of Done (project-wide)](#23-definition-of-done)

---

## 1. Product definition

### 1.1 One-line pitch
An Android app that reads incoming SMS **only from senders you allow**, understands them on-device, optionally reformats or strips them, and posts them to **your own webhook** — while showing you every message it touched and every network request it made, with two hard kill switches.

### 1.2 Goals
| # | Goal | Measured by |
|---|------|-------------|
| G1 | Forward selected SMS to a custom URL reliably | ≥ 99.5 % of matched SMS delivered within 60 s when online (beta telemetry is *local only*, read from the in-app log) |
| G2 | Total transparency | 100 % of outbound requests appear in the network console; OS byte counter vs console byte counter differ by < 5 % |
| G3 | User is always in control | Kill switch takes effect in < 1 s; no SMS read / no request sent after it is engaged |
| G4 | Privacy by default | No third-party SDKs, no analytics, no accounts, no cloud; data encrypted at rest |
| G5 | Simple like a VPN app | A new user sets up a first rule and sends a test request in < 3 minutes |

### 1.3 Non-goals (v1)
- Sending SMS, replying, or being the default SMS app.
- Reading RCS / chat messages (not available to third-party apps).
- iOS (Apple does not allow apps to read SMS).
- Cloud dashboard, user accounts, multi-device sync.
- Cloud AI parsing. All "understanding" is on-device.

### 1.4 Core features (v1 scope)
1. **Sender scope**: *All senders* or *Selected only*, plus an **exclude list** that always wins, plus "never read saved contacts" (on by default).
2. **Rules** (ordered, first match wins): sender(s) → match condition (contains any / regex / any) → parse → format → destination.
3. **Understand**: on-device extractors (amount, currency, account, balance, merchant, OTP, date/time, URL, transaction type).
4. **Format modes**: `Raw`, `Strip` (remove links/promos, mask long numbers), `Template` (user JSON with `{{field}}` placeholders), `Auto` (structured JSON from extracted fields).
5. **Delivery**: HTTPS POST (PUT optional) to allowlisted hosts, custom headers, bearer token, HMAC signature, retries with backoff, durable encrypted outbox.
6. **Messages log**: every SMS seen, with status and a pipeline trace.
7. **Network console**: live log of every outbound request (allowed, denied, held, failed), with redacted headers/body.
8. **Kill switches**: SMS reading kill, network kill, "Stop everything".
9. **Security**: biometric app lock, encrypted DB, Keystore-held secrets, HTTPS-only, optional certificate pinning, OTP/card masking, no backups, FLAG_SECURE.
10. **Retention**: auto-delete logs after N days (default 7).

---

## 2. Platform facts and constraints

| Topic | Fact | Consequence for us |
|---|---|---|
| SMS permissions | `RECEIVE_SMS` (real-time) and `READ_SMS` (inbox history) are *dangerous* runtime permissions and *restricted* on Google Play. | Play listing needs a Permissions Declaration; plan a sideload/GitHub Releases/F-Droid path too (see §18). |
| Receiving | `Telephony.Sms.Intents.SMS_RECEIVED_ACTION` is exempt from implicit-broadcast limits; a manifest receiver is allowed. | Use a manifest-declared receiver protected by `android:permission="android.permission.BROADCAST_SMS"`. |
| Receiver time budget | `onReceive` has ~10 s; process may die after. | `goAsync()` → write to DB → enqueue WorkManager → finish. No network in the receiver. |
| Multipart SMS | One broadcast can carry several PDUs. | Use `Telephony.Sms.Intents.getMessagesFromIntent()` and concatenate by originating address. |
| Background network | Doze / App Standby restrict network. | WorkManager with `NetworkType.CONNECTED`, expedited work for fresh messages, exponential backoff. |
| OEM battery killers | Xiaomi/Oppo/Vivo/Samsung may kill background work. | Onboarding step with per-OEM guidance + deep links; health indicator on Home. |
| Restricted settings (Android 13+) | Side-loaded apps may need "Allow restricted settings" for some sensitive permissions on newer Android versions. | Onboarding detects denial and shows a guided fix. **Verify on Android 15/16 devices in Phase 2.** |
| App-level network block | An app cannot remove its own `INTERNET` permission at runtime. | The network kill switch is enforced in-app at the *single* network chokepoint, cancels all work, and is verified by OS byte counters (§9.6). |
| RCS | Google Messages RCS chats never hit the SMS broadcast. | Document as a limitation; banks/OTPs are SMS, so impact is low. |
| Dual SIM | Intent extras carry subscription id. | Store `subscriptionId`; allow rules to filter by SIM (v1.1). |

**SDK targets**: `minSdk 26` (Android 8.0), `targetSdk`/`compileSdk` = latest stable at kickoff (36 / Android 16 at time of writing).

---

## 3. Architecture

Clean architecture + MVVM with unidirectional data flow.

```
 ┌────────────────────────── UI (Jetpack Compose) ───────────────────────────┐
 │ Home · Messages · Rules · RuleEditor · Senders · NetworkConsole · Security │
 │ Onboarding · Lock                                                          │
 └──────────────▲─────────────────────────────────────────────────────────────┘
                │ StateFlow<UiState>  /  events
 ┌──────────────┴──────────── ViewModels (Hilt) ─────────────────────────────┐
 └──────────────▲─────────────────────────────────────────────────────────────┘
                │ use cases
 ┌──────────────┴──────────── Domain (pure Kotlin, no Android) ──────────────┐
 │ SenderPolicy · RuleMatcher · Extractors · Formatter · Masker · Signer      │
 │ PipelineEngine (deterministic, fully unit-tested)                          │
 └──────▲──────────────────────▲───────────────────────────▲─────────────────┘
        │                      │                           │
 ┌──────┴─────┐   ┌────────────┴─────────┐   ┌─────────────┴──────────────┐
 │ SMS ingest │   │ Data (Room+SQLCipher,│   │ Network (OkHttp, single     │
 │ Receiver + │   │ DataStore, Keystore) │   │ client, Gatekeeper, Logger) │
 │ Reconciler │   └──────────────────────┘   └─────────────▲──────────────┘
 └──────┬─────┘                                            │
        │ enqueue                                          │
        └──────────────► WorkManager: ProcessWorker → DeliveryWorker
```

### 3.1 Runtime flow
1. `SmsReceiver.onReceive` → check **SMS kill switch** (belt-and-braces; the component is also disabled) → `goAsync()`.
2. Build `IncomingSms` (sender, body, timestamp, subId, pdu count) → **SenderPolicy** decides `ALLOW | NOT_SELECTED | EXCLUDED | CONTACT`.
   - If not allowed: persist **metadata only** (sender, time, status). The body is never stored, parsed, or logged.
3. Allowed: persist encrypted `sms_event` with status `RECEIVED` → enqueue unique `ProcessWorker(eventId)` (expedited).
4. `ProcessWorker` runs **PipelineEngine**: Masker → RuleMatcher → Extractors → Formatter → produce `OutboxItem` (or status `NO_MATCH`). Writes a `pipeline_trace`.
5. `DeliveryWorker` drains outbox → `RelayHttpClient` → **NetworkGatekeeper** (kill switch, allowlist, HTTPS) → OkHttp → **NetworkLogger** records everything → update statuses.
6. UI observes Room `Flow`s; everything is live.

### 3.2 Why this shape
- The **domain engine is pure Kotlin** → 100 % deterministic, testable on the JVM in milliseconds, reusable for the in-app "test rule" feature.
- **Exactly one network chokepoint** makes "show all network traffic" and "network kill switch" provable, not best-effort.
- **Outbox pattern** guarantees no SMS is lost across process death, reboots, or kill-switch periods.

---

## 4. Module layout

```
kipta-sms-relay/
├── app/                         # Application, Hilt setup, navigation, manifest, receivers
├── core/
│   ├── model/                   # Pure Kotlin data classes & enums (no Android)
│   ├── domain/                  # PipelineEngine, extractors, matcher, formatter, masker, signer (pure Kotlin/JVM)
│   ├── database/                # Room + SQLCipher, DAOs, migrations, retention
│   ├── datastore/               # Proto/Preferences DataStore: switches, settings
│   ├── security/                # Keystore wrapper, secret vault, biometric helper, DB key manager
│   ├── network/                 # The ONLY module allowed to depend on OkHttp
│   ├── sms/                     # Receiver logic, inbox reconciler, component toggler
│   ├── work/                    # WorkManager workers & scheduling
│   ├── designsystem/            # Theme, colors, typography, components (PowerButton, GlassCard, PillNav…)
│   ├── ui/                      # Shared composables (status chips, empty states, trace view)
│   └── testing/                 # Fakes, fixtures, synthetic SMS corpus
├── feature/
│   ├── onboarding/
│   ├── home/
│   ├── messages/
│   ├── rules/                   # list + editor + test-against-sample
│   ├── senders/
│   ├── console/                 # network console
│   └── security/
├── build-logic/                 # Gradle convention plugins
├── config/detekt/               # detekt config + custom rules (network ban outside core:network)
├── docs/                        # this plan, ADRs, threat model, webhook spec
└── gradle/libs.versions.toml    # version catalog
```

**Dependency rules (enforced by a Gradle check + detekt rule):**
- `core:domain` and `core:model` → no Android, no network.
- Only `core:network` may import `okhttp3.*`, `java.net.HttpURLConnection`, `java.net.URL#openConnection`, `java.net.Socket`.
- `feature:*` → may depend on `core:*` but never on another `feature:*`.

---

## 5. Tech stack

| Concern | Choice | Notes |
|---|---|---|
| Language | Kotlin 2.x (K2) | Coroutines + Flow everywhere |
| UI | Jetpack Compose + Material 3 | Custom VPN-style design system on top of M3 |
| Navigation | Navigation Compose (type-safe routes) | |
| DI | Hilt | `@HiltWorker` for workers |
| DB | Room + SQLCipher for Android (`net.zetetic:sqlcipher-android`) | Passphrase wrapped by Keystore key |
| Settings | DataStore (Proto) | Kill switch state, prefs |
| Background | WorkManager | Expedited + backoff + network constraint |
| HTTP | OkHttp 4/5 (one client) | Interceptors + `EventListener` for console |
| JSON | kotlinx.serialization | Payloads, templates, export |
| Crypto | Android Keystore (AES-GCM, HMAC), `javax.crypto.Mac` | No Tink needed unless we choose it in ADR-004 |
| Biometrics | `androidx.biometric` | `BIOMETRIC_STRONG or DEVICE_CREDENTIAL` |
| Fonts | **Bundled** font files (e.g. Inter / Manrope) | *Not* Google Downloadable Fonts – those use the network |
| Lint | detekt + ktlint + Android Lint | Custom rules for network ban & logging |
| Tests | JUnit5, Kotest assertions, Turbine, MockK, Robolectric, MockWebServer, Compose UI test, Roborazzi screenshots | |
| Build | Gradle KTS, version catalog, convention plugins, R8 | Dependency verification metadata |

**Forbidden dependencies** (CI fails if present): Firebase/Crashlytics/Analytics, any `com.google.android.gms:*` network SDK, ad SDKs, Sentry/Bugsnag, any other HTTP client (Retrofit is allowed only as a thin layer on our OkHttp – decided in ADR-003; default: not used).

---

## 6. Data model

All tables live in a single SQLCipher-encrypted Room database `relay.db`.

### 6.1 `sms_event`
| Column | Type | Notes |
|---|---|---|
| id | TEXT (ULID) PK | |
| sender | TEXT | Raw originating address |
| sender_normalized | TEXT | Upper-case, trimmed, E.164 when numeric |
| received_at | INTEGER (epoch ms) | From PDU timestamp |
| ingested_at | INTEGER | Local clock |
| subscription_id | INTEGER? | SIM slot |
| part_count | INTEGER | Multipart count |
| body | TEXT? | **NULL** for not-selected/excluded/contact senders |
| body_sha256 | TEXT? | For de-duplication |
| status | TEXT enum | `RECEIVED, NOT_SELECTED, EXCLUDED, CONTACT_SKIPPED, NO_MATCH, QUEUED, SENT, HELD, FAILED, DROPPED` |
| status_reason | TEXT? | Human-readable |
| rule_id | TEXT? FK rule | Matched rule |
| source | TEXT enum | `BROADCAST, RECONCILER, TEST` |

### 6.2 `pipeline_trace`
`id, sms_event_id FK, step (READ|FILTER|MATCH|PARSE|MASK|FORMAT|SEND), outcome (OK|SKIP|FAIL), detail TEXT, duration_us INTEGER, at INTEGER`

### 6.3 `rule`
`id, name, position (order), enabled, sender_patterns (JSON list; supports exact and * wildcard), match_type (CONTAINS_ANY|REGEX|ANY), match_values (JSON), case_sensitive, format_mode (RAW|STRIP|TEMPLATE|AUTO), strip_options (JSON), template (TEXT), custom_extractors (JSON: name+regex+group), destination_id FK, created_at, updated_at`

### 6.4 `destination`
`id, label, url, method (POST|PUT), headers (JSON, values referencing secret ids), auth_secret_id?, hmac_secret_id?, pin_sha256 (JSON list)?, timeout_ms, max_attempts, enabled`

### 6.5 `allowed_host`
`host (PK, lower-case), added_at, note` — a destination can only be saved if its host is here.

### 6.6 `sender_filter`
`id, pattern, kind (INCLUDE|EXCLUDE), enabled, created_at`

### 6.7 `outbox`
`id, sms_event_id FK, rule_id, destination_id, payload BLOB (encrypted with DB), content_type, attempt, next_attempt_at, state (PENDING|IN_FLIGHT|SENT|HELD|FAILED|CANCELLED), last_error, idempotency_key`

### 6.8 `network_log`
| Column | Notes |
|---|---|
| id, started_at, finished_at | |
| method, url_redacted, host, path | Query string values redacted |
| outcome | `SENT, DENIED_NOT_ALLOWLISTED, DENIED_CLEARTEXT, BLOCKED_KILL_SWITCH, FAILED_NETWORK, FAILED_TLS, FAILED_PIN, HTTP_ERROR` |
| http_status, duration_ms, dns_ms, connect_ms, tls_ms | From `EventListener` |
| tls_version, cipher, pinned (bool) | |
| request_bytes, response_bytes | |
| request_headers_redacted | `Authorization`, signature, custom secret headers → `••••` |
| request_body_preview | First 4 KB, after masking |
| response_body_preview | First 1 KB |
| outbox_id, sms_event_id | Links to message |
| initiator | `DELIVERY, TEST_REQUEST` — any other value is a bug |

### 6.9 `audit_log`
Every security-relevant change: kill switch toggles, allowlist edits, destination edits, rule edits, secret rotation, data wipe, permission state changes. `id, at, action, detail`.

### 6.10 DataStore `settings.pb`
`sms_reading_enabled, network_enabled, sender_mode (ALL|SELECTED), skip_contacts, mask_otp, mask_cards, retention_days, biometric_lock, hide_bodies, onboarding_done, first_run_at`.

### 6.11 Migrations
Room auto-migrations where possible; every schema version exported to `core/database/schemas/` and covered by a `MigrationTestHelper` test.

---

## 7. The pipeline

### 7.1 Read
- Receiver: `SMS_RECEIVED_ACTION` with priority default; never aborts the broadcast (we are a passive reader).
- **Reconciler** (optional, needs `READ_SMS`): on app start and after the SMS kill switch is turned back *on*, offer "catch up on messages received while paused?" — **off by default**; the user decides. Reads `content://sms/inbox` where `date > last_seen` and applies the same SenderPolicy.
- De-dup: `(sender_normalized, body_sha256, received_at ± 5 s)`.

### 7.2 Filter (SenderPolicy)
Order of evaluation (first decisive result wins):
1. SMS kill switch off → nothing (receiver disabled).
2. Sender matches **EXCLUDE** → `EXCLUDED` (metadata only).
3. `skip_contacts` and sender is a saved contact → `CONTACT_SKIPPED` (needs `READ_CONTACTS`; if not granted, the toggle is shown as "needs permission" and treated as *on* using a numeric-sender heuristic: alphanumeric sender IDs are never contacts).
4. Mode `SELECTED` and sender not in **INCLUDE** → `NOT_SELECTED` (metadata only).
5. Otherwise → allowed.

Patterns: exact (`ACMEBANK`), wildcard (`*BANK*`), numeric normalized to E.164 using the device region.

### 7.3 Mask (before anything else touches the body)
- OTP: 4–8 digit codes near keywords (`OTP`, `code`, `PIN`, `verification`, Bengali equivalents) → `●●●●●●` unless the rule explicitly opts in to "send OTP in clear" (requires biometric confirmation + warning).
- Card/account numbers: sequences of 12–19 digits → keep last 4.
- Masking happens on a copy; the stored body is the original (encrypted) so the user can review it locally.

### 7.4 Match (RuleMatcher)
- Rules evaluated by `position`, **first match wins**.
- `CONTAINS_ANY`: case-insensitive by default, Unicode-normalized (NFC).
- `REGEX`: compiled at save time; validated (compiles, ≤ 512 chars, rejects nested quantifier patterns known to be catastrophic); at runtime run on a `CharSequence` wrapper that checks a deadline (50 ms) → on timeout mark rule `FAIL` in trace and continue to next rule.
- `ANY`: matches any allowed message from the rule's senders.

### 7.5 Understand (Extractors)
Pure-Kotlin, locale-aware extractors returning typed `Field`s with confidence:

| Field | Examples it must handle |
|---|---|
| `amount` + `currency` | `Tk 2,450.00`, `BDT 2450`, `৳২,৪৫০`, `USD 12.50`, `Rs.500` (Bengali digits converted) |
| `type` | debit/credit/withdrawal/deposit/payment/refund/cash-in/cash-out keywords (EN + BN) |
| `account` | `A/C **4821`, `a/c XXXX4821`, `card ending 4821` → `****4821` |
| `balance` | `Avl Bal`, `Available balance`, `Bal:` |
| `merchant` | text after `at`/`to`/`from` up to punctuation |
| `otp` | as §7.3 (masked unless opted in) |
| `datetime` | `21-Oct 15:38`, `21/10/2026`, `2026-10-21 15:38:00`; fallback = received_at |
| `reference` | `TrxID`, `Ref`, `Txn ID` |
| `url` | http(s) links (flagged; dropped by Strip) |
| `phone` | numbers in body |

Custom extractors: user-defined `name + regex + group index`, same ReDoS protection.

The "Understand" step is designed behind an `Extractor` interface so an **optional on-device ML extractor** (e.g. ML Kit / on-device LLM) can be added later (v2), **off by default, never cloud**.

### 7.6 Format
| Mode | Output |
|---|---|
| `RAW` | `{ "sender", "body", "received_at", "sim" }` |
| `STRIP` | as RAW but body cleaned with selected options: remove URLs, drop promo sentences (keyword list: offer, discount, promo, `%` off, cashback…), mask long numbers, collapse whitespace |
| `TEMPLATE` | user JSON with `{{field}}` placeholders; values are **JSON-escaped** by the engine (no injection); missing field → `null` + trace warning; template validated as JSON at save time with sample data |
| `AUTO` | `{ "type", "amount", "currency", "account", "merchant", "balance", "reference", "time", "sender", "raw_masked" }` – only fields found |

Every payload gets an envelope (see §12).

### 7.7 Send
- Create `outbox` row → `DeliveryWorker`.
- Gatekeeper checks (in order): network kill switch → host in allowlist → scheme https → destination enabled.
- Retries: exponential backoff 30 s, 1 m, 2 m, 4 m, 8 m (max attempts configurable, default 5). 4xx (except 408/429) → `FAILED` no retry. 429 → honour `Retry-After`.
- After max attempts → `FAILED`, visible in Messages with a "Retry now" action.
- Kill switch engaged → items become `HELD`; resumed when network is re-enabled.
- Per-destination rate limit: default 30 req/min (token bucket) to avoid flooding.

---

## 8. Kill switches

### 8.1 SMS reading kill switch
When turned **off**:
1. Persist `sms_reading_enabled=false` in DataStore (atomic).
2. `PackageManager.setComponentEnabledSetting(SmsReceiver, DISABLED, DONT_KILL_APP)` → the OS stops delivering SMS broadcasts to us at all.
3. Cancel pending `ProcessWorker`s for messages not yet processed? **No** – already-received messages finish processing (user can also hit network kill). Documented in UI copy.
4. Receiver also checks the flag first (defence in depth for the race window).
5. Audit log entry; Home status card updates instantly.

When turned **on**: re-enable component; optionally prompt the reconciler (§7.1).

### 8.2 Network kill switch
When turned **off**:
1. Persist `network_enabled=false`.
2. `WorkManager.cancelAllWorkByTag("delivery")`; mark `IN_FLIGHT` → `HELD`.
3. `okHttpClient.dispatcher.cancelAll()` and `connectionPool.evictAll()` – in-flight calls are aborted.
4. Gatekeeper interceptor refuses every call with `BLOCKED_KILL_SWITCH` (logged in console, 0 bytes sent).
5. Home + Console show "Network killed" state.

When turned **on**: re-enqueue `HELD` items in original order.

### 8.3 Stop everything
Applies 8.1 + 8.2 atomically; one tap from Home, also available as a **Quick Settings tile** and an **app shortcut** (long-press icon) so it works without unlocking the app UI.

### 8.4 Acceptance tests
- With SMS kill on, sending an SMS via emulator produces **no** `sms_event` row.
- With network kill on, OS `TrafficStats.getUidTxBytes(myUid)` does not increase across 5 queued deliveries.
- Toggle latency (tap → state persisted & component disabled) < 1 s on a low-end device.

---

## 9. Network layer and the network console

### 9.1 Single client
`core:network` exposes `RelayHttpClient` (one `OkHttpClient` singleton). Nothing else in the app has network capability (enforced by §4 rules and the forbidden-dependency check).

### 9.2 Interceptor chain (order matters)
1. **GatekeeperInterceptor** – kill switch, allowlist, https-only, records DENIED/BLOCKED events with 0 bytes and short-circuits with a synthetic failure.
2. **SigningInterceptor** – adds `X-Relay-Timestamp`, `X-Relay-Id`, `X-Relay-Signature`.
3. **ConsoleLoggingInterceptor** – captures redacted request/response previews and sizes.
4. Network: `EventListener.Factory` captures DNS/connect/TLS timings, TLS version, cipher, failures.

### 9.3 TLS
- `network_security_config.xml`: `cleartextTrafficPermitted="false"`, system CAs only (no user CAs in release).
- Optional per-destination SPKI pinning (`CertificatePinner`). UI offers "Pin current certificate" (fetches once, shows fingerprint for the user to confirm – TOFU) or paste pin manually.

### 9.4 Console UI data
- Live list (Room `Flow`, paged), newest first, filter chips: All / Sent / Blocked & held / Failed.
- Row: time · method · status · host/path · duration · bytes.
- Detail: full timeline (DNS → connect → TLS → request → response), redacted headers, body preview, linked SMS.
- Header stats: allowed hosts, requests today, blocked today, bytes up/down today.
- Export: JSON / CSV of the console (local file via SAF, never uploaded).

### 9.5 Redaction rules
`Authorization`, `Cookie`, `X-Relay-Signature`, any header marked secret → `••••`. Query params → `key=••••`. Bodies shown after masking (§7.3).

### 9.6 Independent verification ("Trust but verify")
A card in the console compares:
- **Console total** = Σ request+response bytes logged.
- **OS total** = `TrafficStats.getUidTxBytes/RxBytes(Process.myUid())` delta since app start.
A mismatch > 5 % (beyond TLS/TCP overhead allowance) is shown as a warning. This lets users confirm there is no hidden traffic. Also document how to verify externally with PCAPdroid or a mitmproxy setup (docs/VERIFY_TRAFFIC.md, Phase 6).

---

## 10. Security and privacy design

| Area | Control |
|---|---|
| Data at rest | Room + SQLCipher; 256-bit random DB passphrase generated on first run, wrapped with an AES-GCM key in Android Keystore (StrongBox when available). |
| Secrets | Bearer tokens / HMAC secrets stored encrypted with Keystore; UI never shows them again after save (only "rotate"/"replace"). |
| App lock | BiometricPrompt (`BIOMETRIC_STRONG or DEVICE_CREDENTIAL`) on open and after 5 min in background; separate "reveal bodies" re-auth. |
| Screenshots | `FLAG_SECURE` on Messages, RuleEditor, Console detail, Security. |
| Backups | `android:allowBackup="false"`, `dataExtractionRules` excludes everything (cloud + device transfer). |
| Exported components | Only `SmsReceiver` (protected by `BROADCAST_SMS`), launcher activity, QS tile service (system-bound). Everything else `exported=false`. |
| Logging | No SMS bodies, URLs or secrets in Logcat. R8 rule strips `Log.v/d/i` in release; custom detekt rule bans `Log.*` with non-constant args in `core:sms`, `core:domain`, `core:network`. |
| Clipboard | Copying a body/payload marks clip as sensitive (`ClipDescription.EXTRA_IS_SENSITIVE`, API 33+). |
| Input safety | Templates JSON-escape values; regex ReDoS guard; URL validated & host must be allowlisted. |
| Permissions | Minimal: `RECEIVE_SMS`, `READ_SMS` (only if the user enables catch-up), `INTERNET`, `READ_CONTACTS` (optional), `POST_NOTIFICATIONS`, `USE_BIOMETRIC`. No location, no phone state, no accessibility. |
| Third parties | Zero SDKs that phone home. Dependency verification (`gradle/verification-metadata.xml`). |
| Wipe | "Wipe all local data and keys": deletes DB, DataStore, Keystore aliases, cancels work, disables receiver. |
| Retention | Daily `RetentionWorker` deletes `sms_event`, traces, console logs older than N days (default 7, options 1/7/30/90). |
| Builds | R8 full mode, signed release, reproducible-build effort (§17), SBOM (CycloneDX) attached to releases. |

---

## 11. Threat model

| # | Threat | Vector | Mitigation |
|---|---|---|---|
| T1 | Fake SMS injected by another app | Malicious app sends `SMS_RECEIVED` broadcast | Receiver requires `BROADCAST_SMS` (system only) |
| T2 | OTP / banking data exfiltrated to wrong server | Typo / malicious URL / social engineering | Host allowlist, explicit confirmation when adding hosts, OTP masking on by default, warning UI for "send OTP in clear" |
| T3 | MITM on webhook | Hostile Wi-Fi, rogue CA | HTTPS only, no user CAs, optional SPKI pinning |
| T4 | Webhook spoofing to the user's server | Attacker posts fake events to the user's endpoint | HMAC signature + timestamp + idempotency id; documented server-side verification |
| T5 | Stolen / unlocked phone | Physical access | Biometric lock, encrypted DB, hidden bodies, quick "Stop everything" |
| T6 | Data leakage via backups/screenshots/logcat | Cloud backup, recents screenshot, debug logs | Backups off, FLAG_SECURE, log stripping |
| T7 | Supply-chain compromise | Malicious dependency | Minimal deps, verification metadata, Dependabot, forbidden-deps check, CodeQL |
| T8 | ReDoS / app hang | User or imported regex | Validation + runtime deadline |
| T9 | Hidden network traffic | Library or bug | Single chokepoint, detekt ban, OS byte counter cross-check, external PCAP guide |
| T10 | Replay of queued payloads after long kill | Stale data delivered later | Payload carries `received_at` + `sent_at`; optional "drop held items older than X hours" setting |
| T11 | Malicious rule import | Shared rule file | Import shows full diff and requires host allowlisting separately; imported rules start disabled |

A standalone `docs/THREAT_MODEL.md` will expand this in Phase 6.

---

## 12. Webhook contract

### 12.1 Request
```
POST {destination.url}
Content-Type: application/json; charset=utf-8
User-Agent: KiptaSmsRelay/<version>
X-Relay-Id: 01JAF3K9Q2...           # ULID, idempotency key (stable across retries)
X-Relay-Timestamp: 1792575482       # unix seconds at send time
X-Relay-Signature: sha256=<hex>     # HMAC_SHA256(secret, "<timestamp>.<raw body>")
X-Relay-Attempt: 1
Authorization: Bearer <token>       # optional
```

### 12.2 Body envelope
```json
{
  "version": 1,
  "id": "01JAF3K9Q2...",
  "rule": { "id": "r_txn", "name": "Txn alert" },
  "sms": {
    "sender": "ACMEBANK",
    "received_at": "2026-10-21T15:38:02+06:00",
    "sim": 1
  },
  "format": "auto",
  "data": {
    "type": "debit",
    "amount": 2450.00,
    "currency": "BDT",
    "account": "****4821",
    "merchant": "SHOPMART",
    "balance": 18302.55
  },
  "sent_at": "2026-10-21T15:38:04+06:00"
}
```
- `format = raw|strip` → `data = { "body": "..." }`.
- `format = template` → `data` is the rendered template (any JSON value). A rule option "send template as the whole body (no envelope)" exists for services that require a fixed shape (e.g. Slack/Discord/n8n), in which case signature headers still apply.

### 12.3 Response handling
| Response | Action |
|---|---|
| 2xx | `SENT` |
| 408, 429, 5xx, network error | retry with backoff (honour `Retry-After`) |
| other 4xx | `FAILED`, no retry |
| TLS / pin failure | `FAILED`, no retry, prominent warning |

### 12.4 Server-side verification (documented for users)
1. Reject if `|now − X-Relay-Timestamp| > 300 s`.
2. Compute `HMAC_SHA256(secret, timestamp + "." + rawBody)`; constant-time compare with header.
3. De-duplicate on `X-Relay-Id`.
Reference snippets (Node.js, Python, PHP) will live in `docs/WEBHOOK_SPEC.md` (Phase 4).

---

## 13. UX and design system (VPN-style)

The reference concept is a modern VPN app: airy light-blue gradients, one big circular power control, soft glass cards, pill-shaped bottom navigation, large numerals, playful gradient stat tiles. We map VPN metaphors onto relay concepts:

| VPN metaphor | Relay equivalent |
|---|---|
| Big power button "Connected 00:40:32" | **Master Relay button**: "Relaying · 02:14:09" (uptime since last start) |
| Location cards (Sweden, Brazil…) carousel | **Destination cards** carousel (hooks.example.com, n8n…) with "Active" chip |
| Speed test gauge | **Traffic gauge** in console: requests today / bytes today |
| Download / Upload / Ping / IP tiles | **Sent / Skipped / Held / Blocked** gradient tiles |
| "Secure your connection" onboarding with DNS Leak / VPN Server / Web Server diagram | Onboarding diagram: **SMS → Your phone (Relay) → Your server**, with "Encrypted", "Only your senders", "Only your hosts" chips |
| Pill bottom nav with active label | Same: Home · Messages · Rules · Console · Security |

### 13.1 Tokens
**Color (light)**
| Token | Value | Use |
|---|---|---|
| `bg.gradientTop` | `#CFE6FF` | Screen background top |
| `bg.gradientBottom` | `#F4F8FC` | Screen background bottom |
| `surface` | `#FFFFFF` @ 85 % + blur fallback to solid | Glass cards |
| `ink.primary` | `#0F1720` | Titles |
| `ink.secondary` | `#5A6675` | Body/secondary (≥ 4.5:1 on surface) |
| `brand.primary` | `#2F8CFF` | Power button ring, active nav |
| `brand.deep` | `#1663D6` | Text on light brand surfaces |
| `state.on` | `#2F8CFF` | Running |
| `state.killed` | `#E8623A` / text `#A63A17` | Kill switches engaged (never color-only: label + icon) |
| `tile.sent` | gradient `#BFE0FF → #E8F4FF` | |
| `tile.skipped` | gradient `#E2E6EE → #F5F7FA` | |
| `tile.held` | gradient `#FFE9A8 → #FFF7DA` | |
| `tile.blocked` | gradient `#FFD3C4 → #FFF0EA` | |
| `console.bg` | `#0E1218` | Console panel (dark terminal) |

**Dark theme**: background gradient `#0B1220 → #111827`, surfaces `#172133`, brand `#5AA6FF`. All pairs checked for WCAG AA.

**Typography** (bundled): Display = *Manrope* or *Inter Display* (SemiBold, tight tracking) for big numerals and titles; Body = *Inter*; Mono = *JetBrains Mono* for console, hosts and payloads.
Scale: Display 48/52, H1 32/38, H2 24/30, Title 18/24, Body 15/22, Caption 12/16, Mono 12/18.

**Shape & elevation**: cards 24 dp radius, chips/pills full radius, power button 168 dp circle with 3 concentric soft rings; shadows very soft (y 8, blur 24, 8 % opacity).

**Motion**: power button press → ring ripple + haptic (`HapticFeedbackConstants.CONFIRM`); state change crossfade 250 ms; respect "Remove animations" system setting.

**Iconography**: Material Symbols Rounded (bundled vector drawables). No emoji.

### 13.2 Components (core:designsystem)
`GradientScaffold`, `PowerButton(state: Running|Partial|Stopped)`, `KillSwitchCard`, `GlassCard`, `StatTile(gradient)`, `DestinationCard`, `PillNavBar`, `StatusChip(status)`, `PipelineTrace`, `ConsoleRow`, `ConsoleDetailSheet`, `SegmentedPill`, `SecretField`, `HostChip`, `EmptyState`, `ConfirmDangerSheet`.

### 13.3 Accessibility
- Touch targets ≥ 48 dp; TalkBack labels on all icon buttons; switches expose `Role.Switch` + state description ("SMS reading, on").
- Status never by color only (text + icon).
- Dynamic type up to 200 % without clipping (screens scroll).
- Contrast AA for all text including captions.

### 13.4 Localization
English first; strings externalized from day one; Bengali (`bn`) translation in v1.1. Extractors already handle Bengali digits/keywords.

---

## 14. Screen-by-screen specification

### 14.1 Onboarding (4 steps)
1. **Welcome** – "Relay your SMS. Privately." Diagram SMS → Phone → Your server. CTA "Get started".
2. **How it stays safe** – three cards: *Only senders you pick* · *Only hosts you allow* · *You see every request*. 
3. **Permissions** – explain `RECEIVE_SMS` in plain words → system prompt. If denied/restricted → guided fix screen (including "Allow restricted settings" path).
4. **Reliability** – battery optimization guidance per OEM + notification permission; "Skip for now" allowed.
→ Lands on Home with an empty-state card "Add your first rule".

### 14.2 Home
- Header: app name, lock-state chip "On-device · encrypted".
- **PowerButton** (master): Running / Partially stopped / Stopped; uptime under it.
- Two **KillSwitchCards**: *SMS reading* and *Network access* (title, one-line explanation, switch, ON/OFF text).
- **Destination carousel** (horizontal): each card shows host, rules count, last delivery status.
- **Stat tiles** 2×2: Sent / Skipped / Held / Blocked (today), each tappable → filtered Messages/Console.
- Health row (only when something is wrong): permission revoked, battery optimized, held items, delivery failures.

### 14.3 Messages
- Filter chips: All · Sent · Skipped · Held · Failed; search by sender.
- Rows: sender, time, body preview (hidden behind biometric if setting on; *"Body not opened"* for skipped senders), status chip, rule/destination line.
- Detail sheet: full body (masked view toggle), extracted fields, pipeline trace (Read → Filter → Match → Parse → Format → Send with timings), payload sent, link to console entry, actions: Retry, Re-run rules (dry run), Delete.

### 14.4 Rules
- Ordered list with drag handle, enable switch, summary (From / Match / Format / Send), today's count.
- Top card → Senders screen. FAB "New rule".
- "Test rules" action: paste or pick a past message → shows which rule matches and the payload (dry run, no network).

### 14.5 Rule editor (stepper: Who · Match · Understand · Format · Send)
- **Who**: sender chips + add (with suggestions from recent senders).
- **Match**: segmented Contains / Regex / Any; values as chips; live test against a sample message with highlights.
- **Understand**: table of extracted fields from the sample; add custom field (name + regex + group).
- **Format**: segmented Raw / Strip / Template / Auto; strip option chips; template editor (mono, JSON validation, field autocomplete); live payload preview (dark code panel).
- **Send**: destination picker or create new (URL, method, headers, auth, HMAC secret generate/rotate, pinning, timeout, retries). Host must be allowlisted → inline "Add to allowlist" with confirmation sheet.
- Footer: "Send test request" (goes through the real pipeline & appears in console with initiator `TEST_REQUEST`), "Save".

### 14.6 Senders
- Segmented: All senders / Selected only (with warning copy for "All").
- Selected list with toggles and today's counts; add sender (exact / wildcard), suggestions from recent senders.
- Exclude list (always wins) + "All saved contacts" toggle.

### 14.7 Network console
- Top: VPN-style **traffic gauge** (requests today, arc) + live/killed badge.
- Stat chips: Allowed hosts · Requests today · Blocked · Bytes ↑/↓.
- Filter chips; dark console list (mono); row tap → detail sheet with timeline, headers, body previews.
- "Trust but verify" card (console bytes vs OS bytes).
- Pinned bottom bar: Network kill switch + Export.

### 14.8 Security
Sections: App lock · Outbound requests (HTTPS only – always; sign requests; pin certificates) · Destination allowlist (add/remove hosts) · Data on this phone (masking, skip contacts, encrypted storage, retention, permissions status) · Audit log · Danger zone (Wipe all data and keys).

### 14.9 System surfaces
- **Quick Settings tile**: "Relay" – tap toggles Stop everything / Resume (resume requires unlock).
- **Persistent notification** (optional, off by default): "Relaying · 17 sent today" with a Stop action.
- **App shortcuts**: Stop everything · Network console.

---

## 15. Edge cases and platform quirks

| Case | Handling |
|---|---|
| Multipart SMS | Concatenate PDUs per sender within one broadcast; store `part_count`. |
| Same SMS delivered twice | De-dup window (§7.1). |
| Bengali / Unicode / emoji bodies | NFC normalization; extractors convert Bengali digits. |
| Class-0 / flash SMS | Treated as normal. |
| Dual SIM | Store subscription id; rule filter by SIM in v1.1. |
| Device reboot with HELD/PENDING outbox | WorkManager persists; `BOOT_COMPLETED` not required. |
| Doze / long offline | Items stay `PENDING`, delivered when constraints met; optional "drop older than" setting. |
| Permission revoked in system settings | Detected on resume (`checkSelfPermission`) → Home health warning + audit entry. |
| Webhook returns 3xx | Do **not** follow redirects to other hosts (`followRedirects(false)`); same-host redirects followed only if allowlisted. |
| Very large template output | Cap payload at 64 KB; fail with clear trace message. |
| Regex timeout | Rule skipped for that message; trace shows FAIL; banner on rule. |
| Clock skew | `received_at` from PDU, `sent_at` local; server tolerance documented. |
| App update / migration | Room migrations tested; settings proto versioned. |
| User clears app data | Same as wipe; onboarding restarts. |
| Work profile / multiple users | Each profile is isolated; document. |

---

## 16. Testing strategy

| Layer | Tooling | Target |
|---|---|---|
| Domain engine | JUnit5 + Kotest, property tests for masking/escaping | ≥ 95 % line coverage of `core:domain` |
| Extractor corpus | `core/testing/corpus/*.json` – 200+ **synthetic** SMS (EN/BN, banks, wallets, couriers, OTPs, promos) with expected fields | Regression gate in CI |
| Database | Room in-memory + SQLCipher instrumentation test; migration tests | All migrations |
| Receiver | Robolectric: build PDUs, assert events & statuses | |
| Workers | `WorkManagerTestInitHelper` + MockWebServer: success, retry, 4xx, 429, TLS fail, kill switch mid-flight | |
| Network gate | Unit tests: denied host, cleartext, kill switch → 0 bytes, console row written | 100 % branch |
| UI | Compose UI tests for each screen state; Roborazzi screenshot tests (light/dark, font scale 1.0/2.0) | |
| E2E | Emulator: `adb emu sms send <sender> <text>` → assert delivery at MockWebServer / local test server | Smoke suite on CI (API 30, 34, latest) |
| Security | MobSF static scan, Android Lint security checks, CodeQL, dependency vulnerability scan; manual PCAPdroid/mitmproxy run to confirm only allowlisted traffic | Before each release |
| Performance | Macrobenchmark: cold start < 600 ms on mid device; receiver → DB < 50 ms; pipeline < 20 ms/message | |
| Real devices | Pixel (stock), Samsung (One UI), Xiaomi (HyperOS), one Android 8/9 device | Beta |

---

## 17. CI/CD and repository hygiene

### 17.1 GitHub Actions
- `pr.yml`: ktlint → detekt (incl. custom rules) → Android Lint → unit tests → Roborazzi verify → `assembleDebug` → forbidden-dependency check → upload reports.
- `instrumented.yml` (nightly / label): emulator matrix, E2E SMS smoke tests.
- `codeql.yml`: Kotlin analysis weekly + on PR.
- `release.yml` (tag `v*`): build signed AAB/APK (keystore from GitHub secrets), generate SBOM (CycloneDX), SHA-256 checksums, attach to GitHub Release, draft changelog.

### 17.2 Repo conventions
- Branches: `main` (protected), `feat/*`, `fix/*`, `docs/*`.
- Conventional Commits; PR template with security checklist ("touches network?", "logs user data?", "new permission?").
- `CODEOWNERS`, Dependabot (Gradle + Actions), issue templates (bug, feature, security → private advisory).
- ADRs in `docs/adr/NNN-title.md`. Planned: ADR-001 single network chokepoint · ADR-002 SQLCipher vs Room-only · ADR-003 OkHttp without Retrofit · ADR-004 Keystore direct vs Tink · ADR-005 distribution channel.
- `SECURITY.md` with disclosure process.
- Reproducible builds: pinned JDK & Gradle, `verification-metadata.xml`, deterministic resources; publish reproduction steps.

---

## 18. Distribution and Play policy

1. **GitHub Releases (primary for v1 beta)** – signed APK + checksums + SBOM.
2. **F-Droid** (after v1.0) – fits the no-tracker, open approach.
3. **Google Play** – SMS permissions are restricted; requires the Permissions Declaration Form with a qualifying core use case. Before submitting, re-check the current *SMS and Call Log permissions* policy and exception list (e.g. SMS-based money management, device automation) and prepare: privacy policy, demo video, Data safety form ("data is not collected; sent only to user-configured endpoint"). Treat Play approval as **uncertain** – do not block v1 on it.
4. Privacy policy page (static, in repo `/docs/privacy.md`, published via GitHub Pages).

---

## 19. Phased delivery plan

Estimates assume one senior Android developer. "d" = working days.

### Phase 0 — Foundations (3 d)
| ID | Task | Acceptance criteria |
|---|---|---|
| P0.1 | Create Android project (Kotlin, Compose, minSdk 26), version catalog, convention plugins | `./gradlew build` green |
| P0.2 | Module skeleton per §4 with empty placeholders | Graph matches §4, dependency rules check passes |
| P0.3 | detekt + ktlint + Lint config; custom detekt rules: network-API ban outside `core:network`, logging ban | Rule unit tests pass; violation fails CI |
| P0.4 | Forbidden-dependency Gradle task | Adding Firebase fails the build |
| P0.5 | GitHub Actions `pr.yml`, Dependabot, CODEOWNERS, PR template, SECURITY.md | PR shows all checks |
| P0.6 | ADR-001…005 drafted | Merged in `docs/adr` |

### Phase 1 — Security & storage core (5 d)
| ID | Task | Acceptance criteria |
|---|---|---|
| P1.1 | `KeystoreManager` (AES-GCM keys, StrongBox fallback) | Encrypt/decrypt round-trip instrumented test |
| P1.2 | DB passphrase generation + wrapping | Passphrase never stored in clear; survives restart |
| P1.3 | Room + SQLCipher setup, entities §6, DAOs, schema export | DB file unreadable with plain SQLite |
| P1.4 | DataStore settings proto | Defaults per §6.10 |
| P1.5 | `SecretVault` for tokens/HMAC secrets | Secrets not visible in DB dump |
| P1.6 | Audit log writer | Every settings mutation writes an entry |
| P1.7 | Retention worker | Rows older than N days deleted (test with fake clock) |
| P1.8 | Manifest hardening: backups off, extraction rules, network security config, exported=false | Lint + MobSF baseline clean |

### Phase 2 — SMS ingestion (4 d)
| ID | Task | Acceptance criteria |
|---|---|---|
| P2.1 | `SmsReceiver` with `BROADCAST_SMS` protection, `goAsync`, multipart handling | Robolectric multipart test |
| P2.2 | `SenderPolicy` (include/exclude/contacts/mode) | Unit tests for every branch; skipped senders store **no body** |
| P2.3 | De-dup logic | Duplicate broadcast → one event |
| P2.4 | Component enable/disable (SMS kill switch) | Disabled receiver gets no broadcast on emulator |
| P2.5 | Optional inbox reconciler (`READ_SMS`) | Off by default; catches up only after explicit consent |
| P2.6 | Permission flows incl. restricted-settings detection | Verified on Android 13, 14, 15, 16 sideloaded builds |
| P2.7 | Emulator E2E harness (`adb emu sms send`) | Script in `scripts/` used by CI |

### Phase 3 — Pipeline engine (6 d)
| ID | Task | Acceptance criteria |
|---|---|---|
| P3.1 | `core:model` types: `IncomingSms`, `Rule`, `Field`, `Payload`, `TraceStep`, statuses | Pure Kotlin |
| P3.2 | Masker (OTP, cards) | Property tests: no 12–19 digit run survives |
| P3.3 | RuleMatcher (contains/regex/any, ordering, ReDoS deadline) | Catastrophic regex returns timeout within 60 ms |
| P3.4 | Extractors (§7.5) EN + BN | Corpus ≥ 200 synthetic SMS, ≥ 95 % field accuracy |
| P3.5 | Formatter (raw/strip/template/auto) with JSON escaping | Injection tests (`"`, `\`, `}}`, unicode) pass |
| P3.6 | Envelope builder (§12.2) | Golden JSON tests |
| P3.7 | `PipelineEngine` + trace generation | One call returns result + full trace |
| P3.8 | `ProcessWorker` wiring | Event status transitions correct in DB |

### Phase 4 — Network, outbox, console data (6 d)
| ID | Task | Acceptance criteria |
|---|---|---|
| P4.1 | `RelayHttpClient` singleton, timeouts, no redirects to other hosts | |
| P4.2 | Gatekeeper interceptor (kill, allowlist, https) | Denied calls logged, 0 bytes |
| P4.3 | Signing interceptor (HMAC, timestamp, id, attempt) | Verified by reference server test |
| P4.4 | Console logging interceptor + EventListener timings, redaction | No secret appears in `network_log` (test scans rows) |
| P4.5 | Outbox + `DeliveryWorker` (backoff, 429, 4xx/5xx rules, rate limit) | MockWebServer matrix green |
| P4.6 | Network kill switch: cancel work, dispatcher cancelAll, HELD state, resume | §8.4 tests pass |
| P4.7 | Certificate pinning (manual + TOFU fetch) | Wrong pin → `FAILED_PIN`, no body sent |
| P4.8 | TrafficStats cross-check service | Within 5 % on E2E run |
| P4.9 | `docs/WEBHOOK_SPEC.md` + verification snippets (Node, Python, PHP) + tiny reference receiver in `tools/` | Snippets verify real signed requests |

### Phase 5 — Design system & UI (10 d)
| ID | Task | Acceptance criteria |
|---|---|---|
| P5.1 | Tokens, bundled fonts, light/dark themes | No network font loading; AA contrast check |
| P5.2 | Components (§13.2) with previews + screenshot tests | Roborazzi baselines committed |
| P5.3 | Navigation + PillNavBar + lock gate | Back stack correct; lock after 5 min background |
| P5.4 | Onboarding (§14.1) | First-rule flow < 3 min in usability test |
| P5.5 | Home (§14.2) incl. PowerButton states | Toggle latency < 1 s |
| P5.6 | Messages + detail + trace (§14.3) | Paged list smooth at 10k rows |
| P5.7 | Rules list + reorder + dry-run tester (§14.4) | |
| P5.8 | Rule editor stepper with live preview (§14.5) | Invalid template/regex blocks save with clear error |
| P5.9 | Senders (§14.6) | |
| P5.10 | Network console + detail + gauge + verify card + export (§14.7) | Live updates < 300 ms after request |
| P5.11 | Security screen + allowlist + wipe (§14.8) | Wipe leaves no DB/keys (verified) |
| P5.12 | QS tile, shortcuts, optional notification (§14.9) | Stop everything works from QS while locked |
| P5.13 | Accessibility pass (TalkBack, font 200 %) | Accessibility Scanner: no critical issues |

### Phase 6 — Hardening & security review (4 d)
| ID | Task | Acceptance criteria |
|---|---|---|
| P6.1 | `docs/THREAT_MODEL.md` final | Every threat mapped to a test or control |
| P6.2 | MobSF scan, CodeQL, dependency scan | No high/critical findings open |
| P6.3 | External traffic verification with PCAPdroid/mitmproxy; `docs/VERIFY_TRAFFIC.md` | Only allowlisted hosts observed in 24 h soak |
| P6.4 | Logcat audit on release build | No bodies/URLs/secrets |
| P6.5 | R8 config review, release signing, SBOM, reproducible build check | Two clean builds produce identical APK (or documented diffs) |
| P6.6 | Battery/OEM reliability soak (Samsung, Xiaomi) 72 h | ≥ 99.5 % delivery |

### Phase 7 — Beta (5 d active + 2 weeks soak)
| ID | Task | Acceptance criteria |
|---|---|---|
| P7.1 | v0.9 beta on GitHub Releases | Install guide incl. restricted settings |
| P7.2 | Beta feedback template (issues) | |
| P7.3 | Fix list triage; extractor corpus updated from (anonymized, user-submitted) patterns | |
| P7.4 | Bengali localization | All strings translated |

### Phase 8 — Release v1.0 (3 d)
| ID | Task | Acceptance criteria |
|---|---|---|
| P8.1 | Privacy policy + GitHub Pages | Live |
| P8.2 | v1.0 release with checksums & SBOM | |
| P8.3 | F-Droid metadata submission | Submitted |
| P8.4 | Play Console: declaration form, Data safety, listing (optional) | Submitted / decision recorded in ADR-005 |

### Post-v1 backlog
- Rule filter by SIM, time windows (e.g. only 9–18h).
- Multiple destinations per rule (fan-out).
- Rule import/export (signed JSON) and presets (generic JSON, n8n, Slack, Discord, Telegram bot, Google Sheets via Apps Script, Kipta).
- On-device ML extractor (opt-in).
- Wear OS / widget "Stop everything".
- Encrypted local export/backup with user passphrase.

---

## 20. Timeline summary

| Phase | Duration | Cumulative |
|---|---|---|
| 0 Foundations | 3 d | 3 d |
| 1 Security & storage | 5 d | 8 d |
| 2 SMS ingestion | 4 d | 12 d |
| 3 Pipeline engine | 6 d | 18 d |
| 4 Network & console data | 6 d | 24 d |
| 5 Design system & UI | 10 d | 34 d |
| 6 Hardening | 4 d | 38 d |
| 7 Beta | 5 d + 2 wk soak | 43 d + soak |
| 8 Release | 3 d | ~46 d (~9–10 weeks incl. soak) |

Phases 3 and 5.1–5.2 can run in parallel with a second developer, bringing it to ~6–7 weeks.

---

## 21. Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Play rejects SMS permission | High | Medium | GitHub/F-Droid first; Play optional |
| Restricted settings block sideloaded permission grant | Medium | High | Guided onboarding; test on each Android version early (P2.6) |
| OEM background killing | High | High | Expedited work, OEM guides, health indicator, soak tests |
| Bank SMS formats vary widely | High | Medium | Template/regex escape hatch; corpus grows; dry-run tester |
| User sends OTPs to insecure endpoint | Medium | High | Masking default, allowlist, warnings, HTTPS-only |
| SQLCipher performance on old devices | Low | Low | Benchmarks; indexes; paging |
| Scope creep (presets, ML) | Medium | Medium | Strict v1 scope §1.4; backlog |

---

## 22. Open questions

1. **Licence**: open source (GPL-3.0 / Apache-2.0) or closed? Affects F-Droid.
2. **Relationship with Kipta** (finance app): should a "Kipta" destination preset ship in v1?
3. **Minimum Android version**: keep 26 or raise to 29/31 to simplify?
4. **Persistent notification**: off by default (privacy) or on (reliability on some OEMs)?
5. **Catch-up reconciler** (`READ_SMS`): ship in v1 or defer to reduce permission surface?
6. **Brand**: final app name/icon ("Kipta Relay"?).

---

## 23. Definition of Done

A task is done when:
- Code merged to `main` via PR with green CI (lint, detekt, unit, screenshots).
- New behaviour covered by tests at the right layer.
- No new network path outside `core:network`; no user data in logs.
- Strings externalized; accessibility labels present; light + dark screenshots updated.
- Docs/ADRs updated where behaviour or architecture changed.
- For security-relevant changes: threat model row updated and reviewer sign-off.
