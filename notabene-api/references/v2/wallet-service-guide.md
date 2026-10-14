# Wallet Service Integration Guide

For a wallet service / custody provider that acts as an **agent for another institution** (its customer: an exchange,
VASP or neobank) in v2 Travel Rule transfers. You handle on-chain operations; your customer handles compliance.

Generic mechanics are in the main guide; this file only covers what differs for a wallet service:

- Auth, base URLs, test host: [guide-travel-rule-api.md#authentication](guide-travel-rule-api.md#authentication)
- Webhook verification, envelope, at-least-once / out-of-order delivery:
  [guide-travel-rule-api.md#webhooks](guide-travel-rule-api.md#webhooks)
- Create / authorize / reject / settle / match: [guide-travel-rule-api.md](guide-travel-rule-api.md),
  [api-reference.md#transfers](api-reference.md#transfers)
- Identifier formats (`did:pkh`, CAIP-10, CAIP-19, CAIP-220):
  [guide-travel-rule-api.md#identifier-formats](guide-travel-rule-api.md#identifier-formats)

When your customer adds you as their agent you: provision a wallet for them, determine your side from the agent chain,
confirm address ownership, settle on-chain when authorized, and report settlement.

## Table of Contents

- [Wrapping Transfers for Your Customers](#wrapping-transfers-for-your-customers)
- [Webhook Events](#webhook-events)
- [Determining Your Role](#determining-your-role)
- [Pre-registering Addresses with `custodian`](#pre-registering-addresses-with-custodian)
- [Originator Side (You Send)](#originator-side-you-send)
- [Beneficiary Side (You Receive)](#beneficiary-side-you-receive)
- [Transfer Fields You Need](#transfer-fields-you-need)
- [Preventing Double Settlement](#preventing-double-settlement)
- [Common Pitfalls](#common-pitfalls)
- [Environment Variables](#environment-variables)

---

## Wrapping Transfers for Your Customers

You can offer Travel Rule compliance as a service by wrapping Notabene's Transfers API inside your wallet transaction
API, so your customers comply without integrating Notabene directly. When a customer initiates a withdrawal:

1. `POST /entities/{entityDID}/tx` on their behalf, with **your customer as the primary VASP agent** and **yourself as
   `Custodian` acting `for` your customer**:

```json
{
  "ref": "withdrawal-001",
  "originator": { "@id": "mailto:end-user@example.com" },
  "beneficiary": { "@id": "mailto:recipient@example.com" },
  "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "amount": "100.00",
  "agents": [
    { "@id": "did:web:your-customer-vasp.com", "role": "VASP", "for": "mailto:end-user@example.com",
      "policies": [{ "@type": "REQUIRE_AUTHORIZATION" }] },
    { "@id": "did:web:your-wallet-service.com", "role": "Custodian", "for": "did:web:your-customer-vasp.com" }
  ]
}
```

2. **Wait for authorization.** Your customer's compliance team authorizes or rejects in their Notabene dashboard. You
   get `notification.transferStatusChanged` with `toStatus: "AUTHORIZED"` once your customer and any counterparty
   checks pass. Not broadcasting before this is the critical addition to your existing wallet flow.
3. **Settle on-chain**, then `POST .../tx/{transferId}/settle`.

- **Customer DID:** your customer's entity DID (typically `did:web:customer-domain.com`), obtained at onboarding.
- **Agent chain:** customer → `for` the customer's end-user; you → `for` your customer. The chain decides who does
  compliance (customer) and who settles (you).
- **Deposits:** pre-register your customers' addresses with `custodian` and confirm ownership on their behalf
  ([below](#beneficiary-side-you-receive)).

---

## Webhook Events

| Event | When | What to do |
|---|---|---|
| `notification.transferAgentAdded` | You are added as an agent | Provision wallet, determine role |
| `notification.transferCreated` | A transfer involving you is created | Same as `transferAgentAdded` (may arrive instead of or before it) |
| `tap.requireRelationshipConfirmationRequested` | Address ownership confirmation needed | Confirm only if the address is your client's |
| `tap.requireAuthorizationRequested` | Authorization decision needed | Authorize or reject |
| `notification.transferStatusChanged` → `AUTHORIZED` | All parties authorized | Originator side: settle on-chain |
| `notification.transferStatusChanged` → `SETTLED` | Settlement complete | Beneficiary side: verify funds received |

Payloads: [webhooks.md](webhooks.md).

---

## Determining Your Role

The transfer's `agents` array (GET response shape) is a chain of `{ agent: { "@id" }, for }` entries:

```json
{
  "agents": [
    { "agent": { "@id": "did:web:your-service" }, "for": "did:web:client-vasp" },
    { "agent": { "@id": "did:web:client-vasp" }, "for": "mailto:originator@example.com" }
  ],
  "originator": { "@id": "mailto:originator@example.com" },
  "beneficiary": { "@id": "mailto:beneficiary@example.com" }
}
```

1. Build a map `agent["@id"] → for`.
2. Follow it from your own DID: `you → for → for → ...`
3. Reaching `originator["@id"]` → **originator side** (you send). Reaching `beneficiary["@id"]` → **beneficiary side**
   (you receive).

```
you → client VASP → originator   = originator side (you pay)
you → client VASP → beneficiary  = beneficiary side (you receive)
```

---

## Pre-registering Addresses with `custodian`

Do this on both sides (recommended), for every address you control for a client:

```
POST /entities/{entityDID}/relationships

{
  "from": "did:pkh:eip155:1:0xClientWalletAddress",
  "to": "did:web:client-vasp-did",
  "custodian": "did:web:your-wallet-service.com"
}
```

- `from`: the address (as `did:pkh`) you control for the client
- `to`: your customer's entity DID
- `custodian`: **your own** entity DID

`custodian` is what tells Notabene you manage the address. Without it Notabene identifies the client VASP as owner but
won't add you as an agent on transfers to that address; you'd have to be added manually each time. Pre-registered
addresses also satisfy most relationship confirmations automatically.

---

## Originator Side (You Send)

1. **Provision** a wallet for the client agent DID if none exists; fund it (faucet on testnet) or ensure balance.
2. **Pre-register** addresses ([above](#pre-registering-addresses-with-custodian)).
3. **Authorize** with an empty body: `POST /entities/{entityDID}/tx/{transferId}/authorize` `{}`.
4. **Wait for the transfer-level `AUTHORIZED`** (`notification.transferStatusChanged`), meaning all parties including
   your client's compliance team approved.
5. **Settle:** parse the settlement address (CAIP-10, e.g. `eip155:42431:0xRecipient`) and asset (CAIP-19, e.g.
   `eip155:42431/erc20:0xTokenAddress`), execute ERC-20 `transfer(to, amount)`, wait for confirmation. Claim the
   transfer atomically first ([below](#preventing-double-settlement)).
6. **Report:** `POST .../tx/{transferId}/settle` with `{ "settlementId": "eip155:42431:tx/0xTransactionHash" }`.

---

## Beneficiary Side (You Receive)

1. **Provision** a wallet for the client agent DID.
2. **Pre-register** receiving addresses with `custodian`.
3. **Confirm address ownership** on `tap.requireRelationshipConfirmationRequested`:
   `PATCH /entities/{entityDID}/relationships?from={addressDID}&to={customerDID}` with `{}` (the payload's
   `confirmCallbackUrl`). If the address isn't your client's, don't confirm; the transfer is handled elsewhere.
4. **Authorize** with `{}`.
5. **Reconcile** each on-chain deposit with `GET .../tx/match` ([guide](guide-travel-rule-api.md#4-reconcile-received-transfers)).
   If nothing matches, create the transfer with **yourself as the `VASP` agent `for` the client VASP** and the client
   as beneficiary:

```json
{
  "ref": "your-unique-deposit-reference",
  "originator": { "@id": "did:pkh:eip155:1:0xSenderAddress" },
  "beneficiary": { "@id": "did:web:client-vasp-did" },
  "asset": "eip155:1/erc20:0xTokenAddress",
  "amount": "100.00",
  "agents": [{ "@id": "did:web:your-service.example.com", "role": "VASP", "for": "did:web:client-vasp-did",
               "policies": [{ "@type": "REQUIRE_AUTHORIZATION" }] }],
  "settlementId": "0xTransactionHash"
}
```

6. **Verify** on `toStatus: "SETTLED"` that the funds arrived on-chain (optional, recommended).

---

## Transfer Fields You Need

From `GET /entities/{entityDID}/tx/{transferId}`:

```json
{
  "id": "transfer-uuid",
  "amount": "5000",
  "supportedAssets": ["eip155:42431/erc20:0x...", "USD"],
  "fallbackSettlementAddresses": ["eip155:42431:0x..."],
  "agents": [{ "agent": { "@id": "did:web:..." }, "for": "did:web:..." }],
  "originator": { "@id": "mailto:..." },
  "beneficiary": { "@id": "mailto:..." }
}
```

- `amount`: human-readable (e.g. `"5000"` for $50.00 with 6-decimal tokens)
- `supportedAssets`: CAIP-19 ids the transfer supports (may include fiat such as `"USD"`)
- `fallbackSettlementAddresses`: CAIP-10 settlement addresses, updated as parties provide them
- `agents`: the chain for role determination

---

## Preventing Double Settlement

Duplicate `AUTHORIZED` deliveries, or `AUTHORIZED` arriving while a retry of an earlier event runs, can trigger
settlement concurrently. Use an **atomic claim** so exactly one handler broadcasts:

1. Keep a `status` on your transfer record.
2. Before settling, atomically transition `authorized` → `settling` (e.g. a conditional update).
3. If the transition fails (already `settling` / `settled`), skip.
4. Only the handler that won the claim executes the on-chain transfer.

```
function claimForSettlement(transferId):
  transfer = db.get(transferId)
  if transfer.status != "authorized": return false
  db.update(transferId, { status: "settling" })  // atomic
  return true
```

---

## Common Pitfalls

1. **Authorizing with asset/address in the body.** Wallet services authorize with `{}`; authorization takes no asset or
   address.
2. **Settling on a per-agent signal.** `notification.transferAgentStatusChanged` fires when one agent authorizes; it is
   not enough. Wait for `notification.transferStatusChanged` → `AUTHORIZED`.
3. **Non-idempotent handlers.** Duplicate deliveries cause duplicate records, duplicate on-chain transfers or nonce
   collisions.
4. **Assuming event order.** `AUTHORIZED`, `transferAgentAdded` and `transferCreated` can arrive in any order; check
   current state and act accordingly.
5. **Not pre-registering addresses.** Every incoming transfer then needs manual relationship confirmation.
6. **Omitting `custodian`** when registering addresses: you won't be added automatically to incoming transfers.

---

## Environment Variables

```
NB_CLIENT_ID=<from notabene dashboard>
NB_CLIENT_SECRET=<from notabene dashboard>
NB_ENTITY_DID=did:web:your-service.example.com
NB_API_URL=https://api.notabene.dev
NOTABENE_OAUTH_TOKEN_URL=https://auth.notabene.id/oauth/token
NB_OAUTH_AUDIENCE=https://api.notabene.dev
```

Production: `NB_API_URL` and `NB_OAUTH_AUDIENCE` = `https://api.eu1.notabene.id`.
