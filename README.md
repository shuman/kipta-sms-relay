# Kipta – SMS Relay

**Forward the SMS you choose to your own webhook, privately, and see every byte the app sends.**

A native Android app (Kotlin + Jetpack Compose) that reads incoming SMS **only from senders you allow**, understands them **on-device**, optionally strips or reformats them, and sends them to **your own HTTPS endpoint**. Every message it touches and every network request it makes are shown in the app. Two kill switches let you stop it at any time.

> **Status: planning.** There is no code yet. The full build plan is in [`docs/EXECUTION_PLAN.md`](docs/EXECUTION_PLAN.md).

---

## Why

Bank alerts, wallet notifications, OTPs and delivery updates arrive as SMS. Feeding them into your own tools (n8n, a finance app, a spreadsheet, a server) usually means trusting an opaque app with your most sensitive messages. Kipta – SMS Relay is built the other way round:

- **You choose** which senders it may read. Everything else is never opened.
- **You choose** which hosts it may talk to. Everything else is refused.
- **You can see** each SMS and each request in a log and a live network console.
- **You can stop** reading, sending, or both, with one tap.

## Features (v1 scope)

| | |
|---|---|
| **Sender scope** | Read *all senders* or *selected only*. An exclude list always wins, and saved contacts are skipped by default. |
| **Rules** | Ordered rules, first match wins: sender → match (contains / regex / any) → understand → format → destination. |
| **On-device understanding** | Pulls out amount, currency, account (masked), balance, merchant, transaction type, OTP, date and reference. Handles English and Bengali. No cloud AI. |
| **Formats** | `Raw`, `Strip` (removes links and promos, masks numbers), `Template` (your JSON with `{{fields}}`), `Auto` (structured JSON). |
| **Delivery** | HTTPS only, allowlisted hosts, bearer token, HMAC-SHA256 signature, retries with backoff, and an encrypted outbox that survives reboots. |
| **Messages log** | Each SMS seen, its status (sent, not selected, excluded, no match, held, failed) and a step-by-step pipeline trace. |
| **Network console** | Live log of every outbound request: allowed, denied, held, failed. Shows timings, TLS details, redacted headers and body, and cross-checks the byte count against the OS's own counter. |
| **Kill switches** | *SMS reading* (turns off the receiver at OS level), *Network access* (blocks and cancels all requests and holds the queue), and *Stop everything*. Also available as a Quick Settings tile. |
| **Security** | Fingerprint/face lock, SQLCipher-encrypted database, secrets kept in Android Keystore, OTP and card masking, optional certificate pinning, backups off, screenshots blocked, no third-party SDKs. |

## Design

The look takes its cues from modern VPN apps: soft light-blue gradients, one large circular **power button** for the relay, glass cards, a pill-shaped bottom bar and large numbers.

| VPN idea | In this app |
|---|---|
| Connect button + timer | Relay power button: "Relaying · 02:14:09" |
| Location cards | Destination (webhook) cards |
| Speed test gauge | Traffic gauge in the network console |
| Download / Upload / Ping tiles | Sent / Skipped / Held / Blocked tiles |

The full token set, components and screen specs are in [§13–14 of the plan](docs/EXECUTION_PLAN.md#13-ux-and-design-system-vpn-style).

## Architecture at a glance

```
SMS broadcast ─► SmsReceiver ─► SenderPolicy ─► encrypted DB ─► ProcessWorker
                                                                   │
              PipelineEngine (pure Kotlin): Mask → Match → Understand → Format
                                                                   │
                                   Outbox ─► DeliveryWorker ─► RelayHttpClient
                                                                   │
                   Gatekeeper (kill switch, allowlist, HTTPS) → Signing → Console logger → OkHttp
```

- **One network chokepoint.** Only the `core:network` module can open a connection. CI rules enforce this, which is what makes the console and the network kill switch trustworthy.
- **The pipeline engine is pure Kotlin.** It is deterministic and unit-tested, and the in-app "test rule" dry run uses the same engine.
- **Outbox pattern.** No message is lost to process death, reboots or a kill switch.

## Tech stack

Kotlin 2 · Jetpack Compose + Material 3 · Hilt · Room + SQLCipher · DataStore · WorkManager · OkHttp · kotlinx.serialization · Android Keystore · BiometricPrompt · detekt/ktlint · JUnit5, Robolectric, MockWebServer, Roborazzi · GitHub Actions.

`minSdk 26` · `targetSdk` = latest stable.

## Webhook format (summary)

```http
POST https://your.server/sms
Content-Type: application/json
X-Relay-Id: 01JAF3K9Q2...          # idempotency key
X-Relay-Timestamp: 1792575482
X-Relay-Signature: sha256=<HMAC_SHA256(secret, "<timestamp>.<body>")>
```
```json
{
  "version": 1,
  "id": "01JAF3K9Q2...",
  "rule": { "id": "r_txn", "name": "Txn alert" },
  "sms": { "sender": "ACMEBANK", "received_at": "2026-10-21T15:38:02+06:00", "sim": 1 },
  "format": "auto",
  "data": { "type": "debit", "amount": 2450.00, "currency": "BDT", "account": "****4821", "merchant": "SHOPMART", "balance": 18302.55 },
  "sent_at": "2026-10-21T15:38:04+06:00"
}
```
The full contract, retry rules and server-side verification steps are in [§12 of the plan](docs/EXECUTION_PLAN.md#12-webhook-contract).

## Privacy promise

- No accounts, no analytics, no crash reporters, no ads.
- The app makes no network requests except to the hosts **you** add to the allowlist.
- SMS from senders you have not selected are **never opened**. Only the sender and time are logged.
- All data stays on the phone, encrypted, and is auto-deleted after 7 days by default.

## Platform notes

- **Android only.** iOS does not allow apps to read SMS.
- RCS chat messages cannot be read by third-party apps. Bank and OTP messages arrive as normal SMS.
- Google Play restricts SMS permissions, so the first releases will be on **GitHub Releases** (signed APK, checksums and SBOM), with F-Droid next and Google Play pending policy approval.

## Roadmap

| Phase | Scope | Est. |
|---|---|---|
| 0 | Project setup, modules, CI, lint rules | 3 d |
| 1 | Keystore, encrypted DB, settings, audit, retention | 5 d |
| 2 | SMS receiver, sender policy, SMS kill switch, permissions | 4 d |
| 3 | Pipeline engine: masking, matching, extractors, formatter | 6 d |
| 4 | Network layer, outbox, signing, console data, network kill switch | 6 d |
| 5 | VPN-style design system and all screens | 10 d |
| 6 | Hardening, security review, traffic verification | 4 d |
| 7 | Beta on GitHub Releases | 5 d + soak |
| 8 | v1.0 release | 3 d |

Each task has acceptance criteria in [`docs/EXECUTION_PLAN.md`](docs/EXECUTION_PLAN.md#19-phased-delivery-plan).

## Building (planned)

```bash
git clone <this repo>
cd kipta-sms-relay
./gradlew assembleDebug        # once Phase 0 lands
./gradlew check                # lint, detekt, unit tests
```

## Repository layout (planned)

```
app/  core/{model,domain,database,datastore,security,network,sms,work,designsystem,ui,testing}
feature/{onboarding,home,messages,rules,senders,console,security}
build-logic/  config/detekt/  docs/  scripts/  tools/
```

## License

To be decided (see the plan's open questions).
