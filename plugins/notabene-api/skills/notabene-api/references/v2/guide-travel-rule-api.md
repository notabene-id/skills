# Travel Rule API Guide

The Notabene **v2** Transact API: base URL `https://api.eu1.notabene.id` (EU) or `https://api.us1.notabene.id` (US),
paths under `/entities/{entityDID}/...`.
This guide covers the APIs strictly required to create outgoing transfers, handle incoming transfers and manage the
authorization lifecycle. It is the single home for authentication, identifiers, webhook mechanics and statuses; the
other files link here.

- Every endpoint and schema: [api-reference.md](api-reference.md)
- Every webhook event and payload: [webhooks.md](webhooks.md)
- Wallet services / custody providers acting for another institution: [wallet-service-guide.md](wallet-service-guide.md)
- Embedded UI components: [guide-javascript-sdk.md](guide-javascript-sdk.md)
- Upstream reference: [devx.notabene.id/reference](https://devx.notabene.id/reference)

## Table of Contents

- [Authentication](#authentication)
- [Identifier Formats](#identifier-formats)
- [Webhooks](#webhooks)
  - [Setup](#webhook-setup)
  - [Signature Verification](#signature-verification)
  - [Envelope](#webhook-envelope)
  - [Delivery Guarantees](#delivery-guarantees)
- [Where to Integrate](#where-to-integrate)
- [Outgoing Transfers (Originator)](#outgoing-transfers-originator)
- [Incoming Transfers (Beneficiary)](#incoming-transfers-beneficiary)
- [Transfer Statuses](#transfer-statuses)
- [Webhooks You Must Handle](#webhooks-you-must-handle)

---

## Authentication

1. Generate API credentials in the dashboard (`https://app.eu1.notabene.id/`, Settings > API Credentials).
2. Exchange them for an access token (OAuth2 client credentials):

```
POST https://auth.notabene.id/oauth/token        # US: https://us.auth.notabene.id/oauth/token
Content-Type: application/json

{
  "client_id": "<your_client_id>",
  "client_secret": "<your_client_secret>",
  "grant_type": "client_credentials",
  "audience": "https://api.eu1.notabene.id"   // US: https://api.us1.notabene.id
}
```

Response: `{ "access_token": "eyJhbGc...", "expires_in": 86400, "token_type": "Bearer" }`. The token is valid for
**24 hours**; cache it and refresh before expiry.

3. Send `Authorization: Bearer <access_token>` on every request.

| | EU | US |
|---|---|---|
| Base URL | `https://api.eu1.notabene.id` | `https://api.us1.notabene.id` |
| Token `audience` | `https://api.eu1.notabene.id` | `https://api.us1.notabene.id` |
| Token URL | `https://auth.notabene.id/oauth/token` | `https://us.auth.notabene.id/oauth/token` |

Use the region your onboarding names, and that region's token URL: `auth.notabene.id` answers `403` for the US
audience. v2 has no separate test host: a sandbox entity uses the same base URL. `https://api.notabene.dev` is the v1
test host and accepts only v1 tokens.

Almost every path includes `{entityDID}`: your institution's DID (e.g. `did:web:your-vasp.com`).

---

## Identifier Formats

| Format | Example | Used for |
|---|---|---|
| DID (institution) | `did:web:your-vasp.com` | `entityDID`, VASP / Custodian agents |
| `did:pkh` (address as DID) | `did:pkh:eip155:1:0x742d35Cc...` | `SourceAddress` / `SettlementAddress` agents, relationship `from` |
| `mailto:` | `mailto:customer@example.com` | Customer `@id` (originator / beneficiary) |
| CAIP-19 (asset) | `eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` | `asset` (or a 2–20 char abbreviation like `BTC`) |
| CAIP-10 (account) | `eip155:1:0x742d35Cc...` | `settlementAddress`, address-ownership lookup |
| CAIP-220 (tx) | `eip155:1:tx/0x3edb98...` | `settlementId` (recommended format) |

---

## Webhooks

### Webhook Setup

Configure endpoints in the Notabene dashboard under Settings and subscribe to the events you need (see
[Webhooks You Must Handle](#webhooks-you-must-handle) and the full catalogue in [webhooks.md](webhooks.md)).

### Signature Verification

Webhooks are delivered by [Svix](https://svix.com/) and signed. Verify the signature on the **raw** request body:

```javascript
import { Webhook } from "svix";

const webhook = new Webhook(process.env.SVIX_WEBHOOK_SECRET);

app.post("/webhook", express.raw({ type: "application/json" }), async (req, res) => {
  try {
    const event = webhook.verify(req.body, req.headers); // { message, payload, version }
    await processWebhook(event);
    res.status(200).send("OK");
  } catch (err) {
    res.status(400).send("Invalid signature");
  }
});
```

Svix adds `svix-id`, `svix-timestamp` and `svix-signature` headers.

### Webhook Envelope

Every webhook uses the same envelope:

```json
{
  "message": "notification.transferStatusChanged",
  "payload": {
    "id": "423ae44a-...",
    "for": "did:web:your-domain.com",
    "fromStatus": "OUTGOING",
    "toStatus": "AUTHORIZED"
  },
  "version": "1.0.0"
}
```

| Field | Description |
|---|---|
| `message` | The event type (e.g. `notification.transferStatusChanged`). Not `eventType`, and not repeated inside `payload`. |
| `payload` | All event data. `payload.id` is the transfer id (no top-level `id`); `payload.for` is the DID of the entity the webhook is for. |
| `version` | Currently always `"1.0.0"` |

Callback URLs in payloads (`callbackUrl`, `authorizeCallbackUrl`, ...) are **relative paths**: prepend the API base URL
and otherwise call them as given, even when they use the deprecated `/transfers/...` prefix.

### Delivery Guarantees

- Delivered **at least once** and **in any order** (e.g. `AUTHORIZED`, `transferAgentAdded` and `transferCreated`
  can arrive in any sequence). Make handlers idempotent (de-duplicate on `svix-id` / transfer id + event), check
  current state, and re-read the transfer from the API rather than trusting the payload alone.
- Response codes: `2xx` = processed; `4xx` = not retried; `5xx` = retried with exponential backoff (multiple attempts
  over up to 3 days; failures visible in the Svix dashboard). Return `2xx` quickly and process heavy work async.

---

## Where to Integrate

### Sending crypto (outgoing)

Use [Outgoing Transfers](#outgoing-transfers-originator) wherever your platform sends tokens: exchange withdrawals,
the crypto leg of a fiat onramp, "send" in a custodial wallet or server API, post-trade (OTC) settlement.

**Critical:** create the transfer **before** broadcasting on-chain, and do not send funds until
`notification.transferStatusChanged` arrives with `toStatus: "AUTHORIZED"` (your compliance policies, configured in
the dashboard, are satisfied).

If your policies require extra data from the customer (e.g. beneficiary details), embed the **Withdrawal Assist**
component ([guide-javascript-sdk.md](guide-javascript-sdk.md)) in your withdrawal flow.

### Receiving crypto (incoming)

Use [Incoming Transfers](#incoming-transfers-beneficiary) wherever you receive tokens: exchange deposits, the crypto
leg of a fiat offramp, receiving in a custodial wallet or server API. Usually the Travel Rule message arrives **before**
the funds. Compliance decides via dashboard policies; the developer integrates address-ownership confirmation,
beneficiary-name verification and the authorization decision. Optional SDK components: **Deposit Assist** (collect
missing data from the customer) and **Deposit Request** (enrich the deposit QR code so the sender's VASP can pre-fill
the transfer).

### Delegating settlement to a wallet service

To avoid handling blockchain settlement, add a supported wallet service (custody provider) as an agent acting for you.
It handles address management, on-chain settlement and reporting; you keep compliance (create the transfer, answer
`tap.requirePresentationRequested`, authorize/reject):

```json
"agents": [
  { "@id": "did:web:your-vasp.com", "role": "VASP", "for": "mailto:customer@example.com",
    "policies": [{ "@type": "REQUIRE_AUTHORIZATION" }] },
  { "@id": "did:web:wallet-service.com", "role": "Custodian", "for": "did:web:your-vasp.com" }
]
```

The wallet service receives `notification.transferAgentAdded` and settles only after `AUTHORIZED`. For incoming
transfers it handles address-ownership confirmation and reconciliation. Its side is described in
[wallet-service-guide.md](wallet-service-guide.md).

---

## Outgoing Transfers (Originator)

### 1. Create a Transfer

`POST /entities/{entityDID}/tx`

**To a hosted wallet (known counterparty VASP):**

```json
{
  "originator": { "@id": "mailto:sender@example.com" },
  "beneficiary": { "@id": "mailto:receiver@example.com" },
  "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "amount": "1000",
  "agents": [
    { "@id": "did:pkh:eip155:1:0xSourceAddress", "for": "did:web:your-vasp.com", "role": "SourceAddress" },
    { "@id": "did:web:your-vasp.com", "for": "mailto:sender@example.com", "role": "VASP" },
    { "@id": "did:web:counterparty-vasp.com", "for": "mailto:receiver@example.com", "role": "VASP" },
    { "@id": "did:pkh:eip155:1:0xSettlementAddress", "for": "did:web:counterparty-vasp.com",
      "role": "SettlementAddress" }
  ],
  "ref": "unique-idempotency-key"
}
```

**To an unhosted wallet:** omit the counterparty VASP agent and point the `SettlementAddress` agent's `for` at the
beneficiary (`"for": "mailto:receiver@example.com"`).

After creation Notabene automatically runs agent discovery (if no counterparty was given), checks jurisdictions and
Travel Rule thresholds, applies your default Transaction Authorization Policies, and sends the request via DIDComm to
the counterparty.

| Field | Validation | Description |
|---|---|---|
| `originator` / `beneficiary` | `@id`: 3–255 chars, valid DID | Customer identifiers |
| `asset` | CAIP-19 or 2–20 char abbreviation | e.g. `BTC` |
| `amount` | `^\d+(\.\d+)?$` | Decimal string |
| `agents` | min 1 | Agent chain with `role` and `for` |
| `ref` | `^[a-zA-Z0-9_-]{1,64}$` | Idempotency key (409 on duplicate) |
| `memoTag` | 1–28 printable ASCII | Optional memo/destination tag (XRP, XLM) |

Optional fields (`settlementId`, `transactionValue`, `blockchainAnalyticsAlias`, agent `policies`):
[api-reference.md#post-entitiesentitydidtx](api-reference.md#post-entitiesentitydidtx).

> **Agent roles:** `VASP`, `Custodian`, `SettlementAddress`, `SourceAddress`, `Gateway` (`Unknown` is the default).
> These differ from TAP protocol roles (`OriginatorVASP`, `BeneficiaryVASP`, TAIP-5); the API maps them internally.
> Always use the Notabene values.

### 2. Provide PII When Requested

`tap.requirePresentationRequested` fires when the counterparty's jurisdiction requires Travel Rule data. Its
`originatorPresentationDefinition` / `beneficiaryPresentationDefinition` URLs (e.g.
`https://pd.notabene.id/ivms101/v2/US-3000.json`) say which IVMS101 fields are required. Payload:
[webhooks.md](webhooks.md#tap-policy-requirement-events).

Append PII (Notabene-managed encryption): `POST /entities/{entityDID}/tx/{transferId}/append`

```json
{
  "originator": { "@id": "mailto:sender@example.com" },
  "beneficiary": { "@id": "mailto:receiver@example.com" },
  "ivms101": {
    "originator": {
      "originatorPerson": [{
        "naturalPerson": {
          "name": { "nameIdentifier": [{
            "primaryIdentifier": "Smith", "secondaryIdentifier": "John",
            "naturalPersonNameIdentifierType": "LEGL" }] },
          "dateAndPlaceOfBirth": { "dateOfBirth": "1990-01-15", "placeOfBirth": "Anytown, US" },
          "geographicAddress": [{
            "addressType": "HOME", "streetName": "Main Street", "buildingNumber": "123",
            "postCode": "12345", "townName": "Anytown", "country": "US" }]
        }
      }],
      "accountNumber": ["did:pkh:eip155:1:0xSourceAddress"]
    },
    "beneficiary": {
      "beneficiaryPerson": [{
        "naturalPerson": {
          "name": { "nameIdentifier": [{
            "primaryIdentifier": "Doe", "secondaryIdentifier": "Jane",
            "naturalPersonNameIdentifierType": "LEGL" }] }
        }
      }],
      "accountNumber": ["did:pkh:eip155:1:0xSettlementAddress"]
    }
  }
}
```

Returns `202`. Send only what the presentation definition asks for; requirements vary by jurisdiction. A beneficiary
answering a request for its own customer's data sends just the `ivms101.beneficiary` object (`beneficiaryPerson` +
`accountNumber`). Full structure: [api-reference.md#ivms101](api-reference.md#ivms101).

### 3. Settle the Transfer

After broadcasting (only once `AUTHORIZED`): `POST /entities/{entityDID}/tx/{transferId}/settle`

```json
{ "settlementId": "eip155:1:tx/0x3edb98c24d46d148eb926c714f4fbaa117c47b0c0821f38bfce9763604457c33" }
```

`settlementId` is the on-chain transaction hash (max 255 chars), CAIP-220 `eip155:{chainId}:tx/{txHash}` recommended.

---

## Incoming Transfers (Beneficiary)

### 1. Confirm Address Ownership

**Option A (recommended): pre-register deposit addresses** — `POST /entities/{entityDID}/relationships`

```json
{ "from": "did:pkh:eip155:1:0xYourDepositAddress", "to": "did:web:your-vasp.com" }
```

Pre-registered addresses are confirmed automatically when transfers arrive.

**Option B: respond to the webhook** — on `tap.requireRelationshipConfirmationRequested`, call the payload's
`confirmCallbackUrl` to confirm ownership.

### 2. Verify Beneficiary Name

On `tap.requirePresentationSatisfied` (PII has arrived):

1. `GET /entities/{entityDID}/tx/{transferId}?decrypt=true`
2. Read `beneficiary.beneficiaryPerson.naturalPerson.name.nameIdentifier`.
3. Compare with your KYC records (fuzzy matching, ~90% threshold recommended).
4. Complete both checks (no body) — both must pass before the transfer can be authorized:
   - `POST /entities/{entityDID}/tx/{transferId}/checks/beneficiary-name-matching`
   - `POST /entities/{entityDID}/tx/{transferId}/checks/internal-checks`

If the name does not match, reject with `BENEFICIARY_NOT_FOUND`.

Authorization then happens through your configured policies, or explicitly with
`POST /entities/{entityDID}/tx/{transferId}/authorize`.

### 3. Reject a Transfer

`POST /entities/{entityDID}/tx/{transferId}/reject`

```json
{ "reason": "SANCTION_SCREENING", "comment": "Failed sanctions check" }
```

`comment` is max 500 chars and **required** when `reason` is `OTHER`. Reasons: `COUNTERPARTY_RISK`,
`COUNTERPARTY_DUE_DILIGENCE`, `BLOCKCHAIN_RISK_SCORE`, `SANCTION_SCREENING`, `ASSET_TYPE`, `SUSPICIOUS_TRANSACTION`,
`COUNTERPARTY_POLICIES`, `COUNTERPARTY_REJECTED`, `COUNTERPARTY_NO_RESPONSE`, `CANCELED_BY_INITIATOR`,
`REMOVED_FROM_TRANSFER`, `TRANSFER_PARTICIPANT`, `SOURCE_ADDRESS`, `BENEFICIARY_ADDRESS`, `BENEFICIARY_NOT_FOUND`,
`ORIGINATOR_REJECT_OUTGOING`, `BENEFICIARY_REJECT_INCOMING`, `COMPLIANCE_POLICIES`, `OTHER`.

### 4. Reconcile Received Transfers

Sometimes funds arrive before the Travel Rule message, or you need to match a deposit after the fact:

```
GET /entities/{entityDID}/tx/match?settlement_id={txHash}&settlement_address={destinationAddress}&direction=INCOMING
```

| Parameter | Required | Description |
|---|---|---|
| `settlement_id` | yes | On-chain transaction hash |
| `settlement_address` | yes | Destination address that received funds |
| `settlement_id_index` | no | Vout or log index (UTXO chains, multi-transfer txs) |
| `memo_tag` | no | Memo/tag (XRP, XLM) |
| `direction` | no | `INCOMING` or `OUTGOING` |

`meta.match_strategy`: `strict` (exact match), `broadened` (matched after dropping `settlement_id_index`), `none`.

**No match:** create the transfer yourself so the deposit enters the compliance workflow; Notabene then tries to
identify the originator VASP and request Travel Rule data from it:

```json
{
  "ref": "your-unique-deposit-reference",
  "originator": { "@id": "did:pkh:eip155:1:0xSenderAddress" },
  "beneficiary": { "@id": "mailto:your-customer@example.com" },
  "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "amount": "100.00",
  "agents": [{ "@id": "did:web:your-vasp.com", "role": "VASP", "for": "mailto:your-customer@example.com",
               "policies": [{ "@type": "REQUIRE_AUTHORIZATION" }] }],
  "settlementId": "0xTransactionHash"
}
```

Pattern: on every detected deposit call `match`; link the transfer if found, otherwise create it.

---

## Transfer Statuses

Core lifecycle:

| Status | Description |
|---|---|
| `OUTGOING` | New outgoing transfer created |
| `INCOMING` | New incoming transfer received |
| `AUTHORIZED` | Authorized by all required parties — the only signal to broadcast |
| `REJECTED` | Rejected |
| `SETTLED` | Settled on-chain |
| `REVERTED` | Reverted (terminal) |

Full enum on the Transfer schema: `OUTGOING`, `INCOMING`, `AWAITING-YOURS`, `AWAITING-COUNTERPARTY`, `REJECTED`,
`AUTHORIZED`, `FLAGGED`, `SETTLED`, `FLAGGED-SETTLEMENT`, `REVERTED`, `REVERT-AUTHORIZED`, `REVERT-REJECTED`,
`REVERT-FLAGGED`, `REVERT-REQUESTED`, `FROZEN`, `CLEARED`. `PENDING`, seen in some webhook examples, is **not** a
transfer status; don't branch on it.

Per-agent status (`notification.transferAgentStatusChanged`): `PROCESSING`, `AUTHORIZED`, `REJECTED`, `SETTLED`,
`FAILED`, `REPLACED`, `UNREACHABLE`. An individual agent authorizing is **not** enough to settle; wait for the
transfer-level `AUTHORIZED`.

---

## Webhooks You Must Handle

| Side | Event (`message`) | Action |
|---|---|---|
| Outgoing | `tap.requirePresentationRequested` | Append PII (or POST to `callbackUrl`) |
| Outgoing | `notification.transferStatusChanged` | Broadcast only on `toStatus: "AUTHORIZED"`, then settle |
| Incoming | `tap.requireRelationshipConfirmationRequested` | Confirm via `confirmCallbackUrl` |
| Incoming | `tap.requirePresentationSatisfied` | Verify beneficiary name, complete both checks |
| Incoming | `notification.transferStatusChanged` | Update internal records |

Informational events and all payloads: [webhooks.md](webhooks.md).
