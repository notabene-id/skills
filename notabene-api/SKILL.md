---
name: notabene-api
description: >
  Notabene API integration guide for developers and coding agents, covering all three
  generations of the Notabene Transact (Travel Rule) API: v1 (legacy, /tx/create, vaspDID),
  v2 (api.eu1.notabene.id, /entities/{entityDID}/...) and v2.1, the current platform
  (sandbox.notabene.id, /api/v1/transfers). Use when building, migrating or debugging an
  integration with Notabene, writing code that calls Notabene endpoints or receives its
  webhooks, or asking how Notabene works technically. Triggers on: "Notabene API",
  "Notabene Transact", "Travel Rule API", "originator", "beneficiary", "authorize transfer",
  "settle transfer", "ReadyToSettle", "IVMS101", "entityDID", "vaspDID", "tx/create",
  "notabene auth token", "notabene webhook", "SafeConnect", "WithdrawalAssist",
  "DepositAssist", "DepositRequest", "ConnectWallet", "customer token", "javascript sdk",
  "wallet service", "custody provider", "agent chain", "deposit address".
  Use proactively whenever a developer writes code calling Notabene endpoints.
---

# Notabene API Integration Guide

Notabene Transact (Travel Rule compliance for crypto transfers) has been served by three API generations. They differ
in base URL, paths, payloads, statuses and webhooks, so **the first job is to know which one the integrator is on**.
Then read only that version's guides; never mix calls or field names across versions.

> Flow (payments) is not covered by this skill for now.

---

## Step 0: Which API version?

Infer it from the codebase, config or credentials before asking. Ask the user only when nothing shows it.

| Signal | v1 (legacy) | v2 | v2.1 (current platform) |
|---|---|---|---|
| Base URL | `https://api.notabene.id` (test: `https://api.notabene.dev`) | `https://api.eu1.notabene.id` | `https://sandbox.notabene.id`, `https://eu-a.notabene.id`, `https://us-a.notabene.id` |
| Path shape | `/tx/create`, `/tx/info`, `/tx/update`, `/tx/list?vaspDID=…` | `/entities/{entityDID}/tx`, `/entities/{entityDID}/relationships` | `/api/v1/transfers`, `/api/v1/customers`, `/api/v1/wallets` |
| Institution id | `vaspDID` (often `did:ethr:…`) | `entityDID` in every path (`did:web:…`) | None in the path: the token identifies the institution |
| Transfer statuses | `NEW`, `MISSING_BENEFICIARY_DATA`, `WAITING_FOR_INFORMATION`, `INCOMPLETE`, `NOT_READY`, `SENT`, `ACK`, `ACCEPTED`, `DECLINED`, `REJECTED`, `CANCELLED`, `SAVED` | `AWAITING-YOURS`, `AWAITING-COUNTERPARTY`, `AUTHORIZED`, `REJECTED`, `SETTLED`, … | `Active`, `ReadyToSettle`, `Settled`, `Rejected`, `Canceled`, … |
| Webhooks | Svix, `{ "message", "payload": { "transaction" }, "version" }` with `notification.*` events (registered via `POST /webhook`); deprecated unsigned `txnotification` callback posts the raw transaction | Svix, `{ "message", "payload", "version" }` with `notification.*` / `tap.*` events | Svix, `whsec_` secret set on Integration → Webhooks; `transfer.*.v3` events (`transfer.status_changed.v3`, …) |
| Body casing | camelCase (exceptions: `vasp_did` on `/tx/cancel` and `/v1/addresses`); IVMS uses `nameIdentifierType` | camelCase | snake_case (IVMS-101 keys stay camelCase) |

Beware the name clash: v2.1's paths start with `/api/v1`. That is the **v2.1** platform, not the v1 API.

**New integrations** go on v2.1 unless Notabene has told the institution otherwise. **Existing integrations** stay on
their version; when asked to migrate, read both versions' guides and map each call and event explicitly (the
differences above are not cosmetic: statuses, webhook events and the settle step all change).

---

## v2.1 — the current platform

Read [`references/v2.1/getting-started.md`](references/v2.1/getting-started.md) first, then the guides the path needs.
The API reference served at `/api/openapi.json` on sandbox (and in the dashboard under Integration → Getting Started →
API Reference) is authoritative for field names, enums and permissions.

**1. Find the integration.** Settle before writing code (infer from the codebase; ask only what you cannot see):
direction (withdrawals, deposits or both); who holds the wallets (a wallet provider connected in the dashboard →
wallet-managed, otherwise self-managed); whether the front end runs live checks (→ embedded).
[`choose-your-integration.md`](references/v2.1/choose-your-integration.md) holds the decision tables and the minimal
call/event set per path.

**2. Build the path**, against sandbox, in the guide's order:

| Path | Read |
|---|---|
| Every path | [`getting-started.md`](references/v2.1/getting-started.md) — environments, credential, token, webhook endpoint, conventions, errors |
| The model, for someone new to it | [`overview.md`](references/v2.1/overview.md) — the Travel Rule in payment terms, objects, lifecycle |
| Customers (every path) | [`customers.md`](references/v2.1/customers.md) — register once, ids on transfers, IVMS-101 minimums |
| Deposit addresses (every path that receives) | [`wallets.md`](references/v2.1/wallets.md) — CAIP-10 registration, confirmation requests, self-hosted wallet proofs |
| Withdrawals | [`transact-send.md`](references/v2.1/transact-send.md) |
| Deposits | [`transact-receive.md`](references/v2.1/transact-receive.md) |
| Embedded front end | [`transact-embedded.md`](references/v2.1/transact-embedded.md) |
| Event handling | [`webhooks.md`](references/v2.1/webhooks.md) |

Done when a sandbox transfer reaches `Settled`.

**3. Production.** Walk [`go-live.md`](references/v2.1/go-live.md) against the code and report each unchecked item.

**Invariants worth checking in any v2.1 code:**

- Every transfer create carries a persisted UUID `ref`; retries replay the identical body.
- Every transfer names both customers by id (`originator_id`, `beneficiary_id`); an inline IVMS profile without its
  id is rejected. Customer creation is keyed by `id` and answers `409` when the customer already exists.
- IVMS-101 keys are IVMS's own camelCase (`naturalPersonNameIdentifierType`, not `nameIdentifierType`).
- `POST /api/v1/wallets` takes CAIP-10 only (`eip155:1:0x…`); transfers also accept bare addresses.
- Amounts are whole-unit decimal strings (`"1.5"`, never wei, never a float).
- Withdrawals broadcast only after `ReadyToSettle`, then call settle with the transaction hash.
- Webhooks are verified on the raw body with Svix, de-duplicated on `svix-id`, and re-read state from the API.
- `SettledIncompleteAuthorization` is unfinished; `requires_action: true` means something waits on the integrator.
- Setup happens in the dashboard, not in code: API credential (Integration → API Credentials, secret shown once),
  webhook endpoint and its `whsec_` secret (Integration → Webhooks), wallet provider (Integration → Wallet Providers).
  Ask the user for secrets; never invent, print or commit them.

---

## v2 — `api.eu1.notabene.id`

Read [`references/v2/guide-travel-rule-api.md`](references/v2/guide-travel-rule-api.md) end to end before writing code.

| Need | Guide |
|---|---|
| Outgoing and incoming transfers, PII, authorize/reject, settle | [`guide-travel-rule-api.md`](references/v2/guide-travel-rule-api.md) |
| Every endpoint, request and response schema | [`api-reference.md`](references/v2/api-reference.md) |
| Every webhook event and payload | [`webhooks.md`](references/v2/webhooks.md) |
| Wallet service / custody provider acting as an agent for another institution | [`wallet-service-guide.md`](references/v2/wallet-service-guide.md) |
| Embedding Travel Rule UI (withdrawal, deposit, wallet verification) | [`guide-javascript-sdk.md`](references/v2/guide-javascript-sdk.md) |

**Authentication** — OAuth2 client credentials, token valid 24 hours:
```
POST https://auth.notabene.id/oauth/token
{ "client_id", "client_secret", "grant_type": "client_credentials",
  "audience": "https://api.eu1.notabene.id" }
```
Dashboard: `https://app.eu1.notabene.id/`. Almost every path includes `{entityDID}`, the institution's DID from the
dashboard's Settings.

**Webhooks** — register first with `PUT /entities/{entityDID}/webhook` (manage it via the Svix portal,
`GET …/webhook?generate_portal_url=true`); nothing is sent before that. Delivered by Svix; verify `svix-signature` on
the raw body. Envelope: event type in `message` (not
`eventType`), transfer id in `payload.id`, all data under `payload`.

**Originator lifecycle:**
1. `POST /entities/{entityDID}/tx` — create the transfer **before** broadcasting.
2. On `tap.requirePresentationRequested` → POST the IVMS101 PII to the payload's `callbackUrl` (`.../tx/{id}/append`,
   or `.../policies/{policyId}/presentation` for entity-managed encryption).
3. Wait for `notification.transferStatusChanged` with `toStatus: "AUTHORIZED"`. Never broadcast before it.
4. `POST .../tx/{id}/settle` with `{ "settlementId": "eip155:{chainId}:tx/{txHash}" }`.

**Beneficiary lifecycle:**
1. Register deposit addresses via `POST /entities/{entityDID}/relationships` (recommended; avoids a confirmation per
   transfer). A relationship naming your own DID as `from`, `to` or custodian is confirmed on creation; otherwise
   confirm with `PATCH` on the event's `confirmCallbackUrl`.
2. On `tap.requirePresentationSatisfied` → `GET` the transfer with `?decrypt=true`, run name matching.
3. Complete the checks your entity has configured (`/checks/beneficiary-name-matching`, `/checks/internal-checks`);
   their callback URLs appear in the payload only when configured.
4. Authorize through configured policies or `POST .../tx/{id}/authorize` (`{}`, or `{ "settlementAddress": "<CAIP-10>" }`
   with optional `memoTag`), or reject.
5. Deposits seen on-chain first: find the transfer with `GET .../tx/match`.

**Wallet services (agents acting for another institution):** follow the transfer's `agents` `{ agent, for }` chain
from your DID; reaching `originator` means you send, reaching `beneficiary` means you receive. Register client
addresses with the `custodian` field set to your own DID. Use an atomic claim so duplicate `AUTHORIZED` deliveries
never settle twice.

**JavaScript SDK** — `npm install @notabene/javascript-sdk`. Get a customer token server-side from
`POST /entities/{entityDID}/delegateToken` with `{ entityDID, delegateId }` (valid 1 hour; never ship the API token to
the browser), render `createWithdrawalAssist` / `createDepositAssist` / `createDepositRequest` / `createConnectWallet`,
then convert the result with `componentResponseToTxRequests(response, token, config)`: POST `createTx` to
`/entities/{entityDID}/tx` and `ivms101` to `.../tx/{id}/append`. The response's `txCreate` is v1-shaped; do not send
it to v2.

**Identifiers:** customers `mailto:user@example.com` or `did:pkh:eip155:1:0x…`; addresses `did:pkh:eip155:<chainId>:<address>`;
assets CAIP-19 (`eip155:1/erc20:0xA0b8…eB48`).

---

## v1 — legacy

Read [`references/v1/guide-travel-rule-api.md`](references/v1/guide-travel-rule-api.md). v1 is the original Notabene
API: transactions keyed by `vaspDID`, created with `POST /tx/create` and driven through `/tx/*` verbs. Treat v1 code as
maintenance: fix it in v1 terms, and propose migration to v2.1 rather than extending it with new features.

- **Auth:** `POST https://auth.notabene.id/oauth/token` (client credentials), `audience` `https://api.notabene.id`
  (test: `https://api.notabene.dev`).
- **Safe to send:** the Svix message `notification.processBlockchainTransaction`, fired when the transfer reaches the
  VASP's `statusToProcessBlockchain` setting; `notification.haltBlockchainTransaction` means do not send. Report the
  hash with `POST /tx/update { id, txHash }`.
- **IVMS-101** in v1 uses `nameIdentifierType`; do not apply the v2.1 IVMS key rule to v1 bodies.

---

## Rules for every version

- Never broadcast a withdrawal before the version's "safe to send" signal (v2.1 `ReadyToSettle`, v2 `AUTHORIZED`, v1
  `notification.processBlockchainTransaction`), then report the transaction hash back.
- Webhooks are delivered at least once and can arrive out of order: make handlers idempotent and re-read the transfer
  from the API rather than trusting the payload alone.
- Keep the API access token server-side; front ends use per-customer tokens.
- Do not mix versions: a v2 `entityDID` path, a v2.1 snake_case body and a v1 `vaspDID` query never belong in one call.
- Build against a test environment before production (v2.1 sandbox, a v2 sandbox entity, v1 `api.notabene.dev`).
