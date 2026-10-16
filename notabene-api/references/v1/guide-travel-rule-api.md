# Travel Rule API Guide (v1, legacy)

This guide covers the legacy Notabene **v1** API: hosts `https://api.notabene.id` / `https://api.notabene.dev`,
transfers keyed by `vaspDID` and driven through `/tx/*` verbs. Everything here was checked against the v1 service
source (OpenAPI spec `packages/api/openapi/customer.yml` and the handlers behind it). Items that could not be
verified in code are marked **(unverified)**.

v1 is in maintenance. Fix v1 code in v1 terms; propose migration to v2.1 rather than adding features. Do not mix in
v2 (`/entities/{entityDID}/…`) or v2.1 (`/api/v1/transfers`, snake_case) calls or field names.

## Table of Contents

- [Environments](#environments)
- [Authentication](#authentication)
- [Identifiers and Conventions](#identifiers-and-conventions)
- [Outgoing Transfers (Originator)](#outgoing-transfers-originator)
- [Incoming Transfers (Beneficiary)](#incoming-transfers-beneficiary)
- [Endpoint Reference](#endpoint-reference)
- [Transfer Statuses](#transfer-statuses)
- [Webhooks](#webhooks)
- [IVMS101 and PII](#ivms101-and-pii)
- [Errors](#errors)

---

## Environments

| Environment | API base URL | Token `audience` | Dashboard |
|---|---|---|---|
| Production | `https://api.notabene.id` | `https://api.notabene.id` | `https://app.notabene.id` |
| Test | `https://api.notabene.dev` | `https://api.notabene.dev` | `https://app.notabene.dev` |

Both environments use the same token issuer, `https://auth.notabene.id/`. HTTPS only. JSON request and response
bodies.

---

## Authentication

Every endpoint requires `Authorization: Bearer <access_token>`. The API checks the token's signature against
`https://auth.notabene.id/.well-known/jwks.json` (RS256) and checks the token's issuer and audience.

### 1. API credentials

API credentials (a `clientId` / `clientSecret` pair) belong to a VASP. A VASP admin creates them in the dashboard,
or through the API:

```
POST /auth/credentials/{vaspDID}          body: { "limited"?: boolean, "delegateToGateway"?: boolean }
→ { "clientId": "...", "clientSecret": "..." }
PUT    /auth/credentials/{vaspDID}?clientid=...   rotate
DELETE /auth/credentials/{vaspDID}?clientid=...   delete
```

### 2. Get an access token (OAuth2 client credentials)

```
POST https://auth.notabene.id/oauth/token
Content-Type: application/json

{
  "grant_type": "client_credentials",
  "client_id": "<clientId>",
  "client_secret": "<clientSecret>",
  "audience": "https://api.notabene.id"        // test: "https://api.notabene.dev"
}
```

The response is a standard OAuth2 token response (`access_token`, `expires_in`, `token_type`). The token lifetime is
set in the identity provider, not in this codebase **(unverified)**, so read `expires_in` and refresh before it
expires rather than hard-coding 24 hours.

> The API host also serves `/oauth/token`. That endpoint is a password grant for the dashboard's magic-link login,
> not for server integrations. Use `auth.notabene.id`.

### 3. Customer tokens (front ends)

Keep the API token server-side. To hand a browser a token scoped to one of your customers:

```
POST /auth/customerToken
{ "vaspDID": "did:ethr:0x…", "customerRef": "<your customer id>" }
→ { "access_token", "token_type", "scope" }
```

---

## Identifiers and Conventions

| Thing | Format |
|---|---|
| VASP id | `vaspDID`. The API only enforces the pattern `^did:[a-zA-Z0-9]*:.*$`. Spec examples use `did:ethr:0x…`. Copy yours from the dashboard. |
| Transfer id | `id`, a UUID that Notabene assigns on create |
| Your reference | `transactionRef`: the idempotency key, unique per originator VASP |
| Customer | `originatorDid` / `beneficiaryDid`, created by Notabene from `originatorRef` / `beneficiaryRef` when you leave them out |
| Asset | Symbol string (`"ETH"`, `"BTC"`), or `{ "caip19": "eip155:1/erc20:0x…" }`, or `{ "coingeckoId": "...", "network": "..." }` |
| Amount | String in the asset's **base unit** (wei, satoshi), pattern `^0*[1-9][0-9]*$`. Pass `transactionAssetDecimals` if you need to override the decimals. |
| Addresses | Bare chain addresses in `transactionBlockchainInfo.origin` / `.destination` (not CAIP-10) |

**Casing:** transfer bodies and query parameters are camelCase (`originatorVASPdid`, `transactionBlockchainInfo`).
Note the mixed case of `VASPdid` in body fields versus `vaspDID` in query parameters. Some newer endpoints use
snake_case: `/v1/addresses` takes `vasp_did`, `per_page`, `source_type`, and `/tx/cancel` takes an optional
`vasp_did`.

**Verb style:** state changes are `POST /tx/<verb>`. Some take the transfer id as the query parameter `?id=`
(`approve`, `accept`, `confirm`, `redirect`). Others take it as `"id"` in the JSON body (`update`, `cancel`, `reject`,
`decline`, `notReady`). Most of these return the updated `Transaction`. Check the endpoint table below for each one.

---

## Outgoing Transfers (Originator)

**Rule:** create the transfer in Notabene **before** broadcasting. Broadcast only after the safe-to-send signal
described in step 3.

### 1. Create the transfer

```
POST /tx/create
```

```json
{
  "transactionRef": "a4de3f3f-668f-4844-995a-03ad1b73a54e",
  "transactionAsset": "ETH",
  "transactionAmount": "10000000000000000",
  "originatorVASPdid": "did:ethr:0xORIGINATOR",
  "beneficiaryVASPdid": "did:ethr:0xBENEFICIARY",
  "transactionBlockchainInfo": {
    "origin": "0xa40dfee99e1c85dc97fdc594b16a460717838703",
    "destination": "0x00000000219ab540356cbb839cbe05303d7705fa"
  },
  "originator": {
    "originatorPersons": [{ "naturalPerson": { "name": [{ "nameIdentifier": [
      { "primaryIdentifier": "Baggins", "secondaryIdentifier": "Frodo", "nameIdentifierType": "LEGL" }
    ]}]}}],
    "accountNumber": ["01234567890"]
  },
  "beneficiary": {
    "beneficiaryPersons": [{ "naturalPerson": { "name": [{ "nameIdentifier": [
      { "primaryIdentifier": "Baggins", "secondaryIdentifier": "Bilbo", "nameIdentifierType": "LEGL" }
    ]}]}}],
    "accountNumber": ["0x00000000219ab540356cbb839cbe05303d7705fa"]
  }
}
```

The required fields are `transactionAsset`, `transactionAmount`, `originatorVASPdid` and
`transactionBlockchainInfo.destination`. Optional fields that integrators use:

- `beneficiaryVASPdid`: if you omit it, Notabene tries to resolve the VASP from the destination address. You can
  also pass `beneficiaryVASPname` / `beneficiaryVASPwebsite` / `notificationEmail` for a VASP outside the directory.
- `originatorRef` / `beneficiaryRef` or `originatorDid` / `beneficiaryDid`: your customer references.
- `beneficiaryProof` / `originatorProof`: proof of address ownership for self-hosted wallets. Types:
  `screenshot`, a cryptographic proof, self-declaration, micro-transfer, or other.
- `skipBeneficiaryDataValidation: true`: lets the transfer through when beneficiary data is missing (it lands in
  `MISSING_BENEFICIARY_DATA` instead of failing).
- `travelRuleBehavior: true`: also applies the beneficiary jurisdiction's threshold when Notabene decides whether
  this is a Travel Rule transfer.
- `pii` + `pii_url`, or `encrypted`: see [IVMS101 and PII](#ivms101-and-pii).

**Idempotency:** if a transfer with the same `transactionRef` already exists for this `originatorVASPdid`,
`/tx/create` returns the existing transfer instead of creating a new one. Always send a persisted, unique
`transactionRef`.

**What create does** (from `NotabeneTransactionsWriter.create`):

1. Validates the data against the originator's jurisdiction and works out the `transactionType`:
   `TRAVELRULE`, `BELOW_THRESHOLD`, `NON_CUSTODIAL` or `UNKNOWN`. A validation failure returns `400` with
   `NotabeneTransactionValidationError`. Missing beneficiary data with `skipBeneficiaryDataValidation` gives status
   `MISSING_BENEFICIARY_DATA` and type `UNKNOWN`.
2. Stores the transfer with status `NEW`.
3. Applies your **outgoing rules** (configured in the dashboard). A rule can `APPROVE`, `REJECT` (which cancels
   the transfer), `SEND_TO_INBOX` (leaves it for a human) or `NOT_READY`. If no rule fires, the transfer **stays
   `NEW`**. Internal transfers (same originator and beneficiary VASP) are approved automatically.

### 2. Approve (send) the transfer

If no rule approved it, send it yourself:

```
POST /tx/approve?id=<transferId>
```

This is allowed only from `NEW` (or `INCOMPLETE` for Travel Rule transfers). The result depends on the type:

- `TRAVELRULE`: `SENT` if the beneficiary VASP is claimed on Notabene or has a notification email. Otherwise
  `WAITING_FOR_INFORMATION` (add `notificationEmail` with `/tx/update`).
- `BELOW_THRESHOLD`, `NON_CUSTODIAL` or internal: `SAVED` (nothing to exchange; the transfer is recorded).

After approval, Notabene tries to **auto-confirm**. If the destination is in the beneficiary VASP's address book,
or the beneficiary's address-query webhook answers `owns: true`, the transfer moves `SENT → ACK` with no action
from the beneficiary. If that webhook answers `owns: false`, the transfer is auto-`REJECTED`.

### 3. Wait for the safe-to-send signal

v1 has no single "authorized" status. What is safe depends on a per-VASP setting, `statusToProcessBlockchain`,
which takes one of `CREATED`, `SENT`, `ACK`, `ACCEPTED` or `null`:

```
PUT /tf/vasps/{vaspDID}/settings     { "statusToProcessBlockchain": "ACCEPTED" }
GET /tf/vasps/{vaspDID}/getSettings
```

When this is set and you have registered a [multi-message webhook](#multi-message-webhook-svix), Notabene sends
your VASP (as originator) `notification.processBlockchainTransaction` when the transfer reaches one of these:

- `status` equal to the configured value;
- `SAVED` (non-Travel-Rule transfer), when the setting is anything other than `CREATED`;
- `NOT_READY` (the beneficiary said it is not ready), when the setting is `ACK` or `ACCEPTED`;
- creation itself, when the setting is `CREATED`.

Notabene sends `notification.haltBlockchainTransaction` when the transfer becomes `CANCELLED`, `DECLINED` or
`REJECTED`. It does not send this when the setting is `CREATED`, because the go-ahead was already given at
creation.

**Broadcast on `notification.processBlockchainTransaction` and never after a halt.** Without the setting, there is
no go-ahead message. Your code then has to decide from `status` alone (via webhook or `GET /tx/info`). The strict
reading is to wait for `ACCEPTED`, or `SAVED` for non-Travel-Rule transfers. `ACK` only proves that the address
belongs to the beneficiary VASP.

### 4. Report the transaction hash

After broadcasting:

```
POST /tx/update
{ "id": "<transferId>", "txHash": "0x8f1f…" }
```

`txHash`, `origin` and `destination` can be updated in **any** status. If the beneficiary had already created an
`INCOMPLETE` transfer for the same hash and destination (via `/tx/notify`), Notabene merges that transfer into yours:
theirs becomes `MERGED`.

You can also include `transactionBlockchainInfo.txHash` in `/tx/create` when you record a transfer after the fact.

### Other originator actions

- `POST /tx/cancel` `{ "id", "reason"? }`: allowed from `NEW`, `MISSING_BENEFICIARY_DATA`,
  `WAITING_FOR_INFORMATION`, `SENT` (and `INCOMPLETE` for Travel Rule transfers). Moves the transfer to `CANCELLED`.
- `POST /tx/update` with `originator` / `beneficiary` / `beneficiaryVASPdid` / `…Proof`: changing party data is
  allowed only while the transfer is `INCOMPLETE` or `MISSING_BENEFICIARY_DATA`.
- `POST /tx/validate/full` (same body as create): a dry run that returns `{ isValid, type, errors, warnings,
  beneficiaryVASPdid, beneficiaryAddressType, … }` before you create. `POST /tx/validate` is deprecated.

---

## Incoming Transfers (Beneficiary)

### Receiving

Notabene only shows a transfer to the beneficiary VASP once the originator has sent it. You learn about it from a
webhook (see [Webhooks](#webhooks)), from `GET /tx/list?vaspDID=<you>&txDirection=incoming`, or with
`txGroup=INBOX` for the transfers waiting on you.

### 1. Confirm or reject address ownership (`SENT`)

```
POST /tx/confirm?id=<transferId>                     SENT → ACK      (the address is yours)
POST /tx/reject  { "id", "reason"? }                 SENT → REJECTED (the address is not yours)
POST /tx/notReady { "id", "reason"? }                SENT → NOT_READY (you cannot answer yet)
POST /tx/redirect?id=<transferId>&to=<subsidiaryDID>  route the transfer to a subsidiary VASP
```

You can avoid confirming every transfer by hand:

- **Address book:** register deposit addresses so that Notabene auto-confirms them.
  `POST /address/registerCustomer` `{ "vaspDID", "address", "customerRef", "asset"? }` registers one address.
  `GET /address/upload/{vaspDID}` returns a short-lived URL for uploading an address list in bulk. List or delete
  registered addresses with `GET` / `DELETE /v1/addresses?vasp_did=…`.
  Look up one address with `GET /v1/addresses/address-ownerships/{address}?vasp_did=…&asset=…`.
- **Address-query webhook:** `POST /integrations/{vaspDID}/address-query-webhook` `{ "url", "params" }` returns
  `{ "secret" }`. Notabene then calls your URL as `GET <url><params>&address=…&asset=…` with the header
  `X-Notabene-Signature: base64(HMAC-SHA1(secret, "GET" + fullUrl))` and a 5-second timeout. Answer `200` with
  `{ "owns": true | false, "beneficiaryInfo": … }`; `404` means unknown. `owns: false` auto-rejects the transfer.
- The `autoconfirm` VASP setting (`PUT /tf/vasps/{vaspDID}/settings`): when it is `false`, an address-book match
  is not enough to auto-confirm. A `true` from the webhook still is.

### 2. Accept or decline (`ACK`)

```
POST /tx/accept?id=<transferId>                ACK → ACCEPTED
POST /tx/decline { "id", "reason"? }           ACK → DECLINED
```

Incoming rules in the dashboard can accept or decline transfers automatically. Read the originator data from the
transfer (`GET /tx/info?id=…`) before you decide.

### Deposits seen on-chain first

If a deposit arrives before any Travel Rule message, tell Notabene:

```
POST /tx/notify
{
  "txHash": "0x…", "transactionAmount": "1000000000000000000", "transactionAsset": "ETH",
  "destination": "0x…", "beneficiaryVASPdid": "did:ethr:0xYOU",
  "origin"?: "0x…", "customerRef"?: "...", "beneficiary"?: { … }
}
```

If a matching transfer (by `txHash`, `destination` and `beneficiaryVASPdid`, plus any `matchingParameters` you
pass) is already `SENT`, `/tx/notify` confirms it (`ACK`). Otherwise Notabene creates an `INCOMPLETE` transfer, or a
`SAVED` one for below-threshold and non-custodial deposits. When the originator later reports the same hash, the two
are merged. `GET /tx/query?txHash=…&destination=…&vout=…` finds transfers by on-chain data.

---

## Endpoint Reference

| Method & path | Who | Key input | Notes |
|---|---|---|---|
| `POST /tx/create` | Originator | body: see above | Returns `Transaction`; idempotent on `transactionRef` |
| `POST /tx/validate/full` | Originator | create body | Dry run |
| `POST /tx/approve?id=` | Originator | — | `NEW → SENT / WAITING_FOR_INFORMATION / SAVED` |
| `POST /tx/cancel` | Originator | `{ id, reason?, vasp_did? }` | `→ CANCELLED` |
| `POST /tx/update` | Either | `{ id, txHash?, origin?, destination?, notificationEmail?, originator?, beneficiary?, … }` | Hash updates are allowed in any status |
| `POST /tx/confirm?id=` | Beneficiary | — | `SENT → ACK` |
| `POST /tx/reject` | Beneficiary | `{ id, reason? }` | `SENT → REJECTED` |
| `POST /tx/notReady` | Beneficiary | `{ id, reason? }` | `SENT → NOT_READY` |
| `POST /tx/accept?id=` | Beneficiary | — | `ACK → ACCEPTED` |
| `POST /tx/decline` | Beneficiary | `{ id, reason? }` | `ACK → DECLINED` |
| `POST /tx/redirect?id=&to=` | Beneficiary | subsidiary DID | Route the transfer to a subsidiary |
| `POST /tx/notify` | Beneficiary | on-chain data | `INCOMPLETE`, `SAVED`, or confirms a `SENT` match |
| `GET /tx/info?id=&vaspDID=` | Either | `vaspDID` optional (which of your VASPs you are acting as) | Returns `Transaction` |
| `GET /tx/list?vaspDID=` | Either | `page` (from 0), `resultsPerPage` (max 100), `status`, `txDirection` (`incoming`/`outgoing`/`internal`), `txGroup=INBOX`, `transactionRef`, `q`, `start`/`end`, `wallet`, `sort` (`createdAt:DESC`, …), `decrypt`, `includeActions` | Returns `{ transactions, paginateInfo }` |
| `GET /tx/query` | Either | `txHash`, `destination`, `vout` | Returns `Transaction[]` |

Important `Transaction` response fields: `id`, `status`, `transactionType`, `transactionAsset`,
`transactionAmount`, `originatorVASPdid`, `beneficiaryVASPdid`, `originatorDid`, `beneficiaryDid`,
`transactionBlockchainInfo { txHash, origin, destination }`, `ivms101`, `pii`, `ivms101Encrypted`, `protocol`,
`actions[]`, `comments[]`, `isPayout`.

---

## Transfer Statuses

The public enum (`TransactionStatus` in the spec) has 12 values:

| Status | Meaning |
|---|---|
| `NEW` | Created, not sent yet. Waiting for `/tx/approve` or a rule. |
| `MISSING_BENEFICIARY_DATA` | Created without the beneficiary data the jurisdiction requires |
| `WAITING_FOR_INFORMATION` | Approved, but the beneficiary VASP is unclaimed and has no notification email |
| `SENT` | Delivered to the beneficiary VASP. Waiting for address confirmation. |
| `ACK` | The beneficiary VASP confirmed that it owns the destination address |
| `ACCEPTED` | The beneficiary VASP accepted the transfer (terminal, successful) |
| `DECLINED` | The beneficiary VASP declined the transfer (terminal) |
| `REJECTED` | The beneficiary VASP says the address is not theirs (terminal) |
| `NOT_READY` | The beneficiary VASP is not ready to respond |
| `CANCELLED` | Cancelled by the originator or by an outgoing rule |
| `INCOMPLETE` | Created by the beneficiary from on-chain data (`/tx/notify`), with no originator data yet |
| `SAVED` | Recorded with no Travel Rule exchange (below threshold, non-custodial, internal) |

The code enum also has `MERGED` (a beneficiary-created `INCOMPLETE` transfer folded into the originator's) and
`WAITING_FOR_TR` (set by the internal `/tx/receive`). Handle both as unknown-but-valid. `/tx/list` also accepts
filter-only values (`MISSING_DATA_REQUESTED_LT_5_DAYS`, `MISSING_DATA_REQUESTED_GT_5_DAYS`,
`ORIGINATOR_UNREACHABLE`). These are never stored as a status.

**Main Travel Rule path:**

```
NEW ──approve──▶ SENT ──confirm──▶ ACK ──accept──▶ ACCEPTED
 │                │                 └──decline──▶ DECLINED
 │                ├──reject──▶ REJECTED
 │                └──notReady──▶ NOT_READY
 ├──approve (unclaimed VASP, no email)──▶ WAITING_FOR_INFORMATION
 └──cancel (also from MISSING_BENEFICIARY_DATA / WAITING_FOR_INFORMATION / SENT)──▶ CANCELLED
```

An action from any other status fails with `NotabeneTransactionInvalidStatusError`.

---

## Webhooks

v1 has two webhook mechanisms. They are independent of each other.

### Multi-message webhook (Svix), recommended

Register one URL per VASP:

```
POST   /webhook   { "vaspDID": "did:ethr:0x…", "url": "https://you.example/notabene" }   → signing secret (response body)
GET    /webhook?vaspDID=…                                                                → { "url" } or 404
DELETE /webhook   { "vaspDID": "…" }
```

`POST /webhook` returns the Svix endpoint signing secret as the response body. Store it; you need it to verify
deliveries. Deliveries come from Svix with `svix-id`, `svix-timestamp` and `svix-signature` headers. Verify the raw
body with the Svix library:

```javascript
import { Webhook } from "svix";
const wh = new Webhook(process.env.NOTABENE_V1_WEBHOOK_SECRET);
app.post("/notabene", express.raw({ type: "application/json" }), (req, res) => {
  let evt;
  try { evt = wh.verify(req.body, req.headers); } catch { return res.status(400).end(); }
  // evt.message, evt.payload.transaction
  res.status(200).end();
});
```

Envelope:

```json
{
  "message": "notification.transactionUpdated",
  "payload": { "transaction": { "id": "…", "status": "ACK", "…": "…" } },
  "version": "1.0.0"
}
```

| `message` | When |
|---|---|
| `notification.transactionUpdated` | Every transfer change your VASP is allowed to see |
| `notification.processBlockchainTransaction` | Originator only: safe to broadcast (see [step 3](#3-wait-for-the-safe-to-send-signal)) |
| `notification.haltBlockchainTransaction` | Originator only: transfer is `CANCELLED`, `DECLINED` or `REJECTED`. Do not broadcast. |

By default, Notabene strips `pii.originator`, `pii.beneficiary`, `ivms101.originator` and `ivms101.beneficiary`
from these payloads. Inclusion is controlled by a Notabene-side flag. Fetch PII with `GET /tx/info` when you need it.

### Legacy transaction notification (deprecated)

```
POST /integrations/config/txnotification     { "vaspDID", "callbackUrl" }     (deprecated)
GET  /integrations/txnotification/{vaspDID}
```

On each change, Notabene sends `POST callbackUrl` with the **raw `Transaction` JSON** as the body (no envelope) and
only a `Content-Type: application/json` header. **These deliveries are unsigned**, so treat the body as a hint and
re-read the transfer with `GET /tx/info`. Any non-2xx response is retried, up to 15 attempts with exponential
backoff starting at 10 seconds.

### Who gets notified

The originator VASP is notified of every change. The beneficiary VASP is notified only when the status is
`INCOMPLETE`, `SENT`, `ACK`, `ACCEPTED`, `DECLINED`, `REJECTED` or `NOT_READY`, when a transfer it could already see
is `CANCELLED`, or when a `NON_CUSTODIAL` transfer is `SAVED`. `transactionRef` is removed from the beneficiary's copy.

Deliveries are at-least-once and can arrive out of order. De-duplicate deliveries (on `svix-id` for the Svix
webhook) and act on the state that `GET /tx/info` returns.

---

## IVMS101 and PII

v1 uses IVMS101 for `originator` and `beneficiary` (`originatorPersons[]` / `beneficiaryPersons[]`, each with
`naturalPerson` or `legalPerson`, plus `accountNumber[]`).

> Key name: in v1, the natural-person name type is **`nameIdentifierType`** (`LEGL`, …), not IVMS's
> `naturalPersonNameIdentifierType`. Legal persons use `legalPersonName` / `legalPersonNameIdentifierType`.

There are three ways to send PII on create, update and notify:

1. **Cleartext** `originator` / `beneficiary`. Notabene encrypts them through its PII service and stores a
   reference. Responses fill `ivms101` with the decrypted data for parties allowed to see it.
2. **Pre-encrypted** `pii` (IVMS101-shaped, encrypted values) **together with** `pii_url` (the PII service that can
   decrypt it). Sending only one of the two returns the error `Missing pii and/or pii_url, both should be provided.`
3. **Opaque** `encrypted`: a schema-less encrypted string, stored as `ivms101Encrypted` and passed through. When
   Notabene's GTR validation is enabled, transfers with `targetProtocol: "GTR"` must use it.

PII visibility in responses depends on the caller's relation to the transfer. The originator always sees its own
data. The beneficiary sees originator PII only as the protocol allows.

---

## Errors

Errors use HTTP status codes, with the body `{ "err": { "name", "code", "message", "stack"? } }`.

| Situation | Status / `err.name` |
|---|---|
| Missing or invalid token | `401` `UnauthorizedError` |
| Caller not allowed to act for this VASP | `400`/`403` `UserDIDNotAdminforVASPError` or a permission error |
| Validation failed on create | `400` `NotabeneTransactionValidationError` (`message` lists the missing fields) |
| Unsupported asset | `400` `NotabeneCryptoAssetNotAvailable` |
| No subscription / plan limits | `400` `NotabeneTransactionNoActiveSubscriptionError`, `NotabenePaymentNoCustomerError`, `NotabeneTransactionSunriseLimitReachedError` |
| Action not allowed in the current status | `400` `NotabeneTransactionInvalidStatusError` |
| Unknown transfer id on update | `NotabeneTransactionMissingError` |
| Date range outside the plan's audit-trail window (`/tx/list`) | `400` `{ "error": "Invalid date range. Please check your plan limits." }` |

On status errors, re-read the transfer (`GET /tx/info`) and act from the state it is actually in. Do not retry the
same verb.

---

## Sources (v1 codebase)

- `packages/api/openapi/customer.yml`: public paths, servers, auth scheme
- `packages/api/openapi/paths/tx/*.yml`, `components/schemas/tx/*.yml`: endpoint and schema definitions,
  `TransactionStatus.yml`
- `packages/transactions/src/service.ts`, `transactions.ts`, `writer/writer.ts`, `writer/writerHelper.ts`,
  `writer/writerTxUpdateHelper.ts`, `actions.ts`: handlers, status transitions, idempotency, webhook fan-out
- `packages/travelrule/src/protocols/trlight.ts`: allowed statuses per action for Travel Rule transfers
- `packages/transactions/src/workers/transactions-worker.ts`, `packages/settings/src/types.ts`: the
  process/halt signal and `statusToProcessBlockchain`
- `packages/webhook/src/providers/SVIXWebhook.ts`, `packages/webhook/src/service.ts`: multi-message webhook
- `packages/integrations/src/tx-notification.ts`, `address-query-webhook.ts`: legacy webhook, address-query webhook
- `config/production.yaml`, `config/test.yaml`, `packages/api/src/app.ts`: hosts, issuer, audience, JWT checks
