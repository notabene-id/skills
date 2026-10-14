# Notabene v1 (legacy) Travel Rule API

The v1 API is Notabene's original REST API for Travel Rule transfers between institutions (VASPs). An institution
is identified by a `vaspDID` (usually `did:ethr:0x…`). It creates and drives transactions through verb-style endpoints
(`POST /tx/create`, `/tx/confirm`, `/tx/accept`, …) and gets status changes as Svix-signed webhooks that carry the
whole transaction object. This guide covers only the Travel Rule surface: how to authenticate, the transaction object
and its statuses, the originator and beneficiary flows, webhooks, the customer token for the JS SDK, and the gotchas
that bite integrators. Field names, enum strings and paths come verbatim from Notabene's typed Go client for v1. If you
are starting a new integration, use the current API (see the v2 guides). Use this guide to maintain or debug an
existing v1 integration, or to recognise a v1 integration when you see one.

---

## Table of contents

- [Recognising v1](#recognising-v1)
- [Base URLs and authentication](#base-urls-and-authentication)
- [Errors and response quirks](#errors-and-response-quirks)
- [The transaction object](#the-transaction-object)
- [Transaction statuses](#transaction-statuses)
- [Who may call which verb, and when](#who-may-call-which-verb-and-when)
- [Originator flow (you send crypto)](#originator-flow-you-send-crypto)
- [Beneficiary flow (you receive crypto)](#beneficiary-flow-you-receive-crypto)
- [Deposit first: `/tx/notify`](#deposit-first-txnotify)
- [Address registration and ownership](#address-registration-and-ownership)
- [Webhooks](#webhooks)
- [Customer token for the JavaScript SDK](#customer-token-for-the-javascript-sdk)
- [Reading and listing transactions](#reading-and-listing-transactions)
- [Gotchas](#gotchas)
- [Other endpoints](#other-endpoints)
- [Unverified facts](#unverified-facts)

---

## Recognising v1

You are looking at a v1 integration if you see any of these:

| Signal | v1 | Current API (for contrast) |
| --- | --- | --- |
| Institution identifier | `vaspDID`, typically `did:ethr:0x<40 hex>` (some are `did:web:…`) | `entityDID`, `did:web:…` |
| Path shape | Verb endpoints: `/tx/create`, `/tx/info?id=…`, `/tx/confirm?id=…`, `/tx/accept?id=…`, `/tx/notify`, `/webhook` | Resource paths under `/entities/{entityDID}/…` |
| Scoping | `vaspDID` as a query parameter or body field on almost every call | DID in the path |
| VASP DID field names | `originatorVASPdid`, `beneficiaryVASPdid` (lowercase `did`) | — |
| PII envelope | IVMS 101 split across `originator.originatorPersons[]` and `beneficiary.beneficiaryPersons[]` | — |
| Status strings | `NEW`, `SENT`, `ACK`, `ACCEPTED`, `DECLINED`, … | — |
| Webhook body | `{ "message": "notification.transactionUpdated", "payload": { "transaction": {…} }, "version": … }` | — |
| Error body | `{ "err": { "name", "code", "message", "stack" } }` | — |

Customer identifiers are `originatorDid` / `beneficiaryDid` (lowercase `id`). They are IRIs, not necessarily valid
DIDs: v1 customer DIDs such as `did:email:…` occur.

---

## Base URLs and authentication

### Base URLs

The repository configures these v1 API base URLs:

| Environment | Base URL |
| --- | --- |
| Production | `https://api.notabene.id` |
| QA / staging (EU) | `https://api-qa.eu.notabene.dev` |
| "test" (the v1 counterpart of sandbox) | `https://api.notabene.dev` |

Paths hang directly off the base URL (`https://api.notabene.id/tx/create`). Address endpoints are the exception and
carry their own `/v1` prefix (`/v1/addresses`).

### OAuth client credentials

Every v1 endpoint needs a bearer token; there are no unauthenticated endpoints. Tokens come from the OAuth 2.0
client-credentials grant. The Go client sends `client_id` and `client_secret` as form parameters, plus `audience` when
one is configured:

```
POST <token URL>
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=<clientId>
&client_secret=<clientSecret>
&audience=<audience>
```

Then call the API with `Authorization: Bearer <access_token>`, and refresh the token before it expires.

- **Token URL.** Production Notabene runs on the Auth0 tenant whose custom domain is `https://auth.notabene.id/`, and
  QA uses `https://auth.notabene.dev/`. The token endpoint is the issuer's `/oauth/token`, so production is most
  likely `https://auth.notabene.id/oauth/token`. The repo keeps the actual v1 token URL in a secret, so this is
  inferred, not confirmed.
- **Audience.** The repo's test harness requests a token whose audience equals the API base URL. The repo's own
  production bridge uses a privileged *node-tier* audience (`https://node.api.notabene.id`,
  `https://node.api-qa.eu.notabene.dev`). That audience is for Notabene's own infrastructure, not for integrators.
  Expect a normal institution's audience to be the API base URL (for example `https://api.notabene.id`), but confirm
  it in your Notabene dashboard or onboarding material.
- **Scope of a token.** A credential is minted per institution (`POST /auth/credentials/{vaspDID}`, an admin
  operation that returns `{ "clientId", "clientSecret" }`), and the JWT is scoped to that one `vaspDID`. Reads and
  writes on a transaction your institution is not party to do **not** fail cleanly (see below).

---

## Errors and response quirks

Error responses share one envelope:

```json
{ "err": { "name": "NotabeneTransactionError", "code": 400, "message": "…", "stack": "…" } }
```

`code` usually echoes the HTTP status. Handle these statuses:

| HTTP | Meaning |
| --- | --- |
| 400-class | Validation or illegal status transition. Validation errors can name the missing fields, e.g. `["beneficiaryNameMissing","beneficiaryAccountNumberMissing"]`. |
| 401 | Bad or expired token, or (with some audiences) not a member of the scoped institution. |
| 404 | Real not-found on some endpoints (e.g. `GET /webhook` with no webhook configured, address ownership lookups). |

Quirks that bite:

- **`GET /tx/info` returns `200` with an empty body (or `{}`) when your token cannot see the transaction**, instead of
  a 4xx. Treat an empty body as "not found / not yours".
- **`POST /tx/update` returns `200` even when your token cannot mutate the transaction, and silently does nothing.**
  Verify by re-reading with `/tx/info`.
- **`POST /tx/bulk`** returns either `200` with an array of transactions (finished synchronously) or `202` with an
  array of transaction id strings (queued). Handle both.

---

## The transaction object

Returned by `/tx/create`, `/tx/info`, every verb endpoint, and embedded in every webhook. These are the field names
exactly as they appear on the wire:

| Field | Type | Notes |
| --- | --- | --- |
| `id` | UUID | Notabene's transaction id. Use it on every later call. |
| `transactionRef` | string | Your own reference, echoed back on every webhook. Use it to correlate. |
| `status` | string | See [statuses](#transaction-statuses). |
| `transactionType` | string | `BELOW_THRESHOLD`, `NON_CUSTODIAL`, `TRAVELRULE`, `UNKNOWN`. |
| `transactionAsset` | string | Asset **symbol** (`ETH`, `USDC`, `BTC`). The create request also accepts `{ "caip19": "…" }` or `{ "coingeckoId", "network" }` in its place. |
| `transactionAmount` | string | **Integer amount in the asset's smallest unit** (wei for ETH, satoshi for BTC). `"1000000000000000000"` = 1 ETH. |
| `chargedQuantity` | number | |
| `amountInLocalCurrency` | object | `{ "amountInLocalCurrency": <JSON number>, "currency": "USD" }`, the originator jurisdiction's value. |
| `beneficiaryAmountInLocalCurrency` | object | The same shape, for the beneficiary jurisdiction. |
| `originatorDid` / `beneficiaryDid` | string (IRI) | Customer identifiers; may be `did:email:…` and other non-DID IRIs. |
| `originatorVASPdid` / `beneficiaryVASPdid` | DID | The two institutions. |
| `transactionBlockchainInfo` | object | `{ "txHash", "origin", "destination" }`. `destination` is required on create. |
| `ivms101` | object | IVMS 101 envelope (see below). |
| `ivms101Encrypted` | string | Encrypted IVMS payload, when the PII is end-to-end encrypted. |
| `pii` | object | IVMS 101-shaped PII block (the same shape as `ivms101`). |
| `protocol` | string | `TRLight`, `TRP`, `OpenVASP`: the protocol negotiated with a counterparty off the Notabene network. |
| `protocolData` | object | `{ "trLightData": { "notificationEmails": [ … ] } }` |
| `originatorProof` / `beneficiaryProof` | object | Wallet ownership proof (see below). |
| `actions` | array | Audit log: `{ "hash", "issuerDID", "action", "actionData" }`. |
| `comments` | array | `{ "vaspDID", "createdAt", "createdBy": { "userDID", "userName" }, "text" }` |
| `isPayout` | bool | Payments flag; ignore for plain Travel Rule. |

### IVMS 101 envelope

```json
{
  "originator": {
    "originatorPersons": [ { "naturalPerson": { … } } ],
    "accountNumber": ["<institution account id>"]
  },
  "beneficiary": {
    "beneficiaryPersons": [ { "legalPerson": { … } } ],
    "accountNumber": ["<institution account id>"]
  },
  "originatingVASP": { "originatingVASP": { "legalPerson": { … } } },
  "beneficiaryVASP": { "beneficiaryVASP": { "legalPerson": { … } } },
  "transferPath": { "transferPath": [ { "intermediaryVASP": { … }, "sequence": 1 } ] },
  "payloadMetadata": { "transliterationMethod": ["othr"] }
}
```

Each person is `{ "naturalPerson": … }` **or** `{ "legalPerson": … }`, never both.

- `naturalPerson`: `name[]` (each with `nameIdentifier[]`, `localNameIdentifier[]` and `phoneticNameIdentifier[]` of
  `{ primaryIdentifier, secondaryIdentifier, nameIdentifierType }`), `geographicAddress[]`, `nationalIdentification`,
  `customerIdentification`, `dateAndPlaceOfBirth { dateOfBirth (YYYY-MM-DD), placeOfBirth }`, `countryOfResidence`.
- `legalPerson`: `name { nameIdentifier[] of { legalPersonName, legalPersonNameIdentifierType } }`,
  `geographicAddress[]`, `customerNumber`, `nationalIdentification`, `dateOfRegistration`, `countryOfRegistration`.
- Address: `addressType` (`HOME`, `BIZZ`, `GEOG`), `streetName`, `buildingNumber`, `buildingName`, `postCode`,
  `townName`, `countrySubDivision`, `addressLine[]`, `country` (ISO 3166 alpha-2), and so on.
- Codes: `nameIdentifierType` takes `LEGL`, `ALIA`, `BIRT`, `MAID` or `MISC`. `legalPersonNameIdentifierType` takes
  `LEGL`, `SHRT` or `TRAD`. `nationalIdentifierType` takes `ARNU`, `CCPT`, `RAID`, `DRLC`, `FIIN`, `TXID`, `SOCS`,
  `IDCD`, `LEIX` or `MISC`.
- `primaryIdentifier` is the **last name**; `secondaryIdentifier` holds the first names.
- `accountNumber` is an institution's account identifier for the customer. Do not confuse it with the on-chain
  address in `transactionBlockchainInfo`, although many integrations reuse the address there.

### Ownership proofs (`originatorProof`, `beneficiaryProof`)

The `type` field selects the variant:

| `type` | Fields |
| --- | --- |
| `eip-191`, `eip-712`, `bip-137` | `proof` (signature), `attestation` (signed message), `address`, `status` |
| `screenshot` | `url` |
| `self-declaration` | `attestation`, `did`, `address`, `confirmed` (bool), `status` |
| `microtransfer` | `proof`, `did`, `address`, `chain`, `destination`, `amountSubunits`, `status` |

A beneficiary proof matters for self-hosted (unhosted) destinations: `/tx/validate` can report
`beneficiaryOwnershipProofMissing`.

---

## Transaction statuses

These are the exact enum strings in the client. The meanings come from the client and the Notabene specs that
exercise v1; where a meaning is inferred, the table says so.

| Status | Meaning |
| --- | --- |
| `SAVED` | Stored but not sent as a Travel Rule transfer. A below-threshold create parks here and never fires a `SENT` webhook. |
| `NEW` | Created, awaiting the **originator's** approval. `/tx/approve` (or an auto-approve rule on the originator institution) moves it to `SENT`. |
| `MISSING_BENEFICIARY_DATA` | Created without the beneficiary data the Travel Rule needs. Can still be approved, and accepts `/tx/update` data. |
| `WAITING_FOR_INFORMATION` | Waiting for party data. The repo treats it like `INCOMPLETE` (data may still be added). Exact trigger not visible in the repo. |
| `INCOMPLETE` | A deposit notification (`/tx/notify`) without the originator's data. The originator institution supplies it with `/tx/update`. |
| `SENT` | Delivered to the beneficiary institution. Waiting for it to confirm the address (`/tx/confirm`), reject it (`/tx/reject`) or redirect it (`/tx/redirect`). |
| `ACK` | The beneficiary institution confirmed it controls the destination address. Waiting for its decision (`/tx/accept` or `/tx/decline`). |
| `ACCEPTED` | The beneficiary institution accepted. Terminal success. |
| `REJECTED` | The beneficiary refused at `SENT` (`/tx/reject`). Terminal. |
| `DECLINED` | The beneficiary declined after review (`/tx/decline`). Terminal. |
| `CANCELLED` | The originator cancelled (`/tx/cancel`). Terminal. |
| `NOT_READY` | The beneficiary marked the transaction not ready (`/tx/notReady`, with a reason). Inferred from the endpoint; follow-up transitions are not visible in the repo. |

Happy path for an originator-created transfer: `NEW → SENT → ACK → ACCEPTED`. A deposit notification takes
`INCOMPLETE → SENT → ACK → ACCEPTED`. Treat any status you don't recognise (the specs mention `MERGED`, for example) as
non-actionable. Log it rather than crash on it.

---

## Who may call which verb, and when

The legacy status map (as cited in the repo's specs, plus behaviour observed against the real v1 stack):

| Verb | Caller | Allowed from | Result |
| --- | --- | --- | --- |
| `POST /tx/approve?id=` | originator | `NEW`, `MISSING_BENEFICIARY_DATA` | `SENT` |
| `POST /tx/confirm?id=` | beneficiary | `SENT` only | `ACK` |
| `POST /tx/reject` | beneficiary | `SENT` only | `REJECTED` |
| `POST /tx/redirect?id=&to=` | beneficiary | `SENT` only | Status unchanged; `beneficiaryVASPdid` becomes `to` |
| `POST /tx/accept?id=` | beneficiary | `ACK` (the map lists `SENT`, `SAVED` and `INCOMPLETE` too, but an accept against `SENT` is **refused** in practice) | `ACCEPTED` |
| `POST /tx/decline` | beneficiary | `ACK`, `SENT`, `SAVED`, `INCOMPLETE` | `DECLINED` |
| `POST /tx/cancel` | originator | most non-terminal statuses | `CANCELLED` |
| `POST /tx/update` (`txHash`) | either | any status | Records the hash |
| `POST /tx/update` (party data) | depends | `INCOMPLETE`, `MISSING_BENEFICIARY_DATA` (also `SAVED`) | Re-validated; may move the status (see the deposit flow) |

Institutions can configure **rules** that act without a human. An originator's *outgoing approve* rule advances
`NEW → SENT` during `/tx/create`, and a beneficiary's *autoConfirm* rule advances `SENT → ACK` a few seconds later when
the destination resolves through that institution's own address book. Rules run asynchronously, so poll or wait for
the webhook rather than assuming a status right after a call.

---

## Originator flow (you send crypto)

```
(optional) POST /tx/validate/full ──► POST /tx/create ──► [NEW] ──► POST /tx/approve (or auto-approve rule)
                                                                         │
                                         [SENT] ◄────────────────────────┘
                                           │  beneficiary: /tx/confirm      (or /tx/reject → REJECTED, /tx/redirect)
                                         [ACK]
                                           │  beneficiary: /tx/accept       (or /tx/decline → DECLINED)
                                       [ACCEPTED] ──► broadcast on-chain ──► POST /tx/update { id, txHash }
```

### 1. Validate (optional)

`POST /tx/validate/full` dry-runs a create. It takes the same body as `/tx/create` plus `validatePartyFields`
(`BENEFICIARY`, `ORIGINATOR` or `ALL`):

```json
{
  "transactionAsset": "ETH",
  "transactionAmount": "1000000000000000000",
  "originatorVASPdid": "did:ethr:0xaaaa…",
  "transactionBlockchainInfo": { "destination": "0xBeneficiaryAddress" },
  "validatePartyFields": "BENEFICIARY"
}
```

The response is `{ isValid, type, beneficiaryAddressType, addressSource, beneficiaryVASPdid, beneficiaryVASPname,
errors[], warnings[], beneficiary, id }`:

- `type` is the `transactionType`, which tells you whether the Travel Rule applies.
- `beneficiaryAddressType` is `HOSTED`, `UNHOSTED` or `UNKNOWN`.
- `addressSource` says how Notabene resolved the destination: `ADDRESS_BOOK`, `ADDRESS_GRAPH`, `ADDRESS_HASH`,
  `CHAINALYSIS`, `ELLIPTIC` or `UNKNOWN`.
- `beneficiaryVASPdid` is set when Notabene resolved the destination address to an institution.

`POST /tx/validate` is the older, **deprecated** variant. Its body is flatter: `transactionAsset`, `destination`,
`transactionAmount`, `originatorVASPdid`, `originatorEqualsBeneficiary` (required bool), `beneficiaryVASPdid`,
`beneficiaryName`, `beneficiaryAccountNumber`, `beneficiaryAddress`, `beneficiaryProof`. Its `errors` are drawn from
`beneficiaryNameMissing`, `beneficiaryAccountNumberMissing`, `beneficiaryAddressMissing` and
`beneficiaryOwnershipProofMissing`.

### 2. Create

`POST /tx/create`:

```json
{
  "transactionRef": "withdrawal-8841",
  "transactionAsset": "ETH",
  "transactionAmount": "1000000000000000000",
  "originatorVASPdid": "did:ethr:0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "beneficiaryVASPdid": "did:ethr:0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
  "transactionBlockchainInfo": {
    "origin": "0xOriginAddress",
    "destination": "0xBeneficiaryAddress"
  },
  "originator": {
    "originatorPersons": [{
      "naturalPerson": {
        "name": [{ "nameIdentifier": [{
          "primaryIdentifier": "Baggins", "secondaryIdentifier": "Frodo", "nameIdentifierType": "LEGL"
        }] }],
        "dateAndPlaceOfBirth": { "dateOfBirth": "1980-01-01", "placeOfBirth": "Shire" },
        "geographicAddress": [{
          "addressType": "HOME", "streetName": "Bagshot Row", "buildingNumber": "1",
          "townName": "Hobbiton", "country": "SG"
        }]
      }
    }],
    "accountNumber": ["0xOriginAddress"]
  },
  "beneficiary": {
    "beneficiaryPersons": [{
      "naturalPerson": {
        "name": [{ "nameIdentifier": [{
          "primaryIdentifier": "Bolson", "secondaryIdentifier": "Bilbo", "nameIdentifierType": "LEGL"
        }] }]
      }
    }],
    "accountNumber": ["0xBeneficiaryAddress"]
  }
}
```

Required fields: `transactionAsset`, `transactionAmount`, `originatorVASPdid` and `transactionBlockchainInfo`
(with `destination`). The other fields are optional:

| Field | Purpose |
| --- | --- |
| `beneficiaryVASPdid` | Omit it, or send `beneficiaryVASPname` / `beneficiaryVASPwebsite`, when the counterparty is unknown. |
| `originatorDid`, `beneficiaryDid`, `originatorRef`, `beneficiaryRef` | Your customer identifiers. |
| `transactionAssetDecimals` | The asset's precision. |
| `customAssetPrice` | `{ priceUSD, decimals, price, currency }`. Only for assets Notabene cannot price; see [gotchas](#gotchas). |
| `originatorProof`, `beneficiaryProof` | Wallet ownership proofs. |
| `pii`, `encrypted` | PII supplied as an IVMS-shaped block or as an encrypted string. |
| `protocol`, `targetProtocol` | `targetProtocol: "GTR"` routes the transfer to a GTR counterparty. |
| `notificationEmail` | Used for TRLight. |
| `skipBeneficiaryDataValidation` | Skips beneficiary data validation. |
| `travelRuleBehavior` | Travel Rule behaviour flag. |
| `network` | The asset's network. |
| `onBehalfOfMyself` | The customer sends to their own account at another institution. |
| `blockchainTransactionTimestamp` | When the transfer happened on-chain. |

Validation rules observed against the real v1 stack:

- An **above-threshold** transfer must carry beneficiary persons. Without them the create fails with
  `400 ["beneficiaryNameMissing","beneficiaryAccountNumberMissing"]`. A `beneficiary` block without
  `beneficiaryPersons` fails with a schema error.
- **Every** originator person on an above-threshold transfer needs a name, an account number, and a national id,
  date and place of birth, or geographic address. A name alone yields `originatorGeographicAddressMissing` or
  `originatorDateAndPlaceOfBirthMissing`.
- A **below-threshold** create parks at `SAVED` and is never sent.
- Party DIDs must match `^did:[a-zA-Z0-9]*:.*$`.

The response is the transaction, usually at `NEW`, or already at `SENT` when your institution has an auto-approve rule.

### 3. Approve

If the transaction is at `NEW`, send it with `POST /tx/approve?id=<id>` (no body). Skip this if your auto-approve rule
already moved it to `SENT`; check `status` in the create response or the webhook.

### 4. Wait for the beneficiary institution

Watch `notification.transactionUpdated` webhooks (or poll `GET /tx/info?id=…`):

| Status | What to do |
| --- | --- |
| `SENT` | Wait. The beneficiary hasn't confirmed the address yet. |
| `ACK` | The beneficiary confirmed the address; its decision is pending. |
| `ACCEPTED` | Go: the beneficiary institution accepted the transfer. |
| `REJECTED` / `DECLINED` | Stop: do not broadcast. |
| `SENT` with a changed `beneficiaryVASPdid` | The transfer was redirected to another institution. Re-evaluate. |

**When is it safe to broadcast?** The v1 lifecycle has no separate "authorized" status: `ACCEPTED` is the
beneficiary's positive decision. Notabene's own bridge treats `ACCEPTED` as the counterparty's authorization and only
then settles. Broadcasting earlier (at `SENT` or `ACK`), or for a transfer whose counterparty is unknown or
unreachable, is a risk-policy decision for your compliance team. Nothing in the API blocks it.

### 5. Record the on-chain hash

After broadcasting, `POST /tx/update` with the hash. Updating `txHash` is allowed in any status:

```json
{ "id": "6f1c…-uuid", "txHash": "0x9c1d…" }
```

`/tx/update` also takes `destination`, `origin`, `beneficiaryVASPdid`, `beneficiaryVASPname`, `originator`,
`beneficiary`, `originatorProof`, `beneficiaryProof`, `pii`, `isNonCustodial`, `comment`, `originatorNameHash`,
`beneficiaryNameHash` and `skipBeneficiaryDataValidation`. Use them to complete data on a transaction that is still
collecting it.

> **Never send empty strings for fields you don't mean to change.** An empty `"beneficiaryVASPdid": ""` on
> `/tx/update` clears that field. Omit unset keys.

### 6. Cancel

The originator terminates with `POST /tx/cancel`:

```json
{ "id": "6f1c…-uuid", "vasp_did": "did:ethr:0xaaaa…", "reason": "customer cancelled withdrawal" }
```

Note the snake_case `vasp_did`. A cancel after `ACCEPTED` is not a legal transition. Re-read the status first.

### `/tx/updateTR`

`POST /tx/updateTR` takes `{ transactionId, destinationAddress, assetType, amount, pii, originatorVASPdid,
beneficiaryVASPdid }`, all required except `pii`. Note the different field names (`transactionId`, `assetType`,
`amount`). The client exposes it, but nothing in the repo exercises it, so its exact semantics are unverified. Prefer
`/tx/update` for the documented flows.

---

## Beneficiary flow (you receive crypto)

```
webhook (status SENT, you are beneficiaryVASPdid) ──► GET /tx/info?id=…
    ├─ you control the destination?        yes ──► POST /tx/confirm?id=…   ──► [ACK]
    │                                       no  ──► POST /tx/reject {id, reason} ──► [REJECTED]
    │                                       wrong institution ──► POST /tx/redirect?id=…&to=<did> (still SENT)
    └─ [ACK] compliance review ──► POST /tx/accept?id=…        ──► [ACCEPTED]
                                └► POST /tx/decline {id, reason} ──► [DECLINED]
                                   (or POST /tx/notReady {id, reason} ──► [NOT_READY])
```

1. **Receive the webhook.** The transaction arrives as `notification.transactionUpdated` with `status: "SENT"` and
   your DID in `beneficiaryVASPdid`. The webhook copy may omit PII, so read the full object with
   `GET /tx/info?id=<id>`.
2. **Confirm the address** with `POST /tx/confirm?id=<id>` (no body). This moves `SENT → ACK` and asserts that your
   institution controls `transactionBlockchainInfo.destination`. **Confirm is a separate step from accepting.** Send
   it only when you have checked the address belongs to your customer. An autoConfirm rule does this for you when the
   address is in your address book (see [address registration](#address-registration-and-ownership)).
3. **Read the originator's PII after confirming.** v1 escrows the originator's PII. Before the beneficiary confirms,
   the originator block in webhooks and `/tx/info` can come back empty (`{}`) for the beneficiary. Re-read
   `/tx/info` after `ACK`.
4. **Decide:**
   - `POST /tx/accept?id=<id>` (no body), valid from `ACK`. An accept sent at `SENT` is refused. The status read can
     lag the confirm that produced `ACK` by a few seconds, so retry a refused accept briefly.
   - `POST /tx/decline` with `{ "id": "…", "reason": "…" }`, after review.
   - `POST /tx/notReady` with `{ "id": "…", "reason": "…" }`, to signal you can't decide yet.
5. **Alternatives at `SENT` only:**
   - `POST /tx/reject` with `{ "id": "…", "reason": "unknown destination address" }`.
   - `POST /tx/redirect?id=<id>&to=<did>` hands the transaction to another institution. The status stays `SENT`;
     only `beneficiaryVASPdid` changes. Only the **beneficiary** can be redirected; there is no originator redirect.

Verb examples:

```
POST /tx/confirm?id=6f1c…-uuid
POST /tx/accept?id=6f1c…-uuid
POST /tx/redirect?id=6f1c…-uuid&to=did:ethr:0xcccc…
```

```json
{ "id": "6f1c…-uuid", "reason": "sanctions hit on beneficiary name" }
```

Each verb returns the updated transaction.

### Bulk actions

`POST /tx/bulk` applies one action to every transaction between two institutions, optionally filtered by addresses:

```json
{
  "action": "accept",
  "originatorVASPdid": "did:ethr:0xaaaa…",
  "beneficiaryVASPdid": "did:ethr:0xbbbb…",
  "destinationAddress": "0xBeneficiaryAddress",
  "reason": "…"
}
```

`action` takes `accept`, `approve`, `cancel`, `confirm`, `decline`, `reject`, `notReady` or `redirect`. `redirect`
also needs `beneficiaryVASPdidTo`. The response is `200` with transactions, or `202` with ids (see
[errors](#errors-and-response-quirks)).

### `/tx/receive`

`POST /tx/receive` takes `{ "transactionId", "address", "asset", "originatorVASPdid", "beneficiaryVASPdid" }`, all
required. The client exposes it, but the repo doesn't exercise it, so its exact effect (for example, whether it
creates a beneficiary-side record for a deposit already seen on-chain) is unverified. For "the deposit landed before
any Travel Rule data", the flow the repo verifies end to end is `/tx/notify`, below.

---

## Deposit first: `/tx/notify`

`/tx/notify` is a **deposit notification**, not a settlement notice. The beneficiary institution sees funds arrive
on-chain with no Travel Rule transfer yet, and tells Notabene so the originator institution can supply its data.

```json
{
  "txHash": "0x9c1d…",
  "transactionAsset": "ETH",
  "transactionAmount": "1000000000000000000",
  "beneficiaryVASPdid": "did:ethr:0xbbbb…",
  "destination": "0xBeneficiaryAddress",
  "origin": "0xOriginAddress",
  "customerRef": "cust-123"
}
```

- Required: `transactionAmount`, `transactionAsset`, `beneficiaryVASPdid` and `destination`.
- Optional:

  | Field | Purpose |
  | --- | --- |
  | `txHash` | The deposit's on-chain hash. Omit it rather than send `""`: the field must match `^[^\s]+$`. |
  | `origin` | The sending address. |
  | `originatorVASPdid` | Names the originator institution when you know it (see below). |
  | `transactionAssetDecimals` | The asset's precision. |
  | `customerRef` | Your customer reference. |
  | `network` | The asset's network. |
  | `targetProtocol` | Routes the notification to another protocol (`GTR`). |
  | `encrypted` | Encrypted PII. |
  | `beneficiaryNameHash` | |
  | `matchingParameters` | Which fields to match on: `transactionAmount`, `transactionAsset`, `origin`. |
  | `id` | Pre-assigns the transaction id. Only a Notabene-internal caller may do this; integrators omit it. |
- **What happens:**
  - **Matches an existing transaction** (for example by `txHash` + `destination` + `beneficiaryVASPdid`): v1 links to
    it and auto-confirms it to `SENT`/`ACK`. It belongs to whichever flow created it.
  - **No match:** a new transaction at `INCOMPLETE`. Notabene resolves the originator institution from the `origin`
    address when it can. `originatorVASPdid` isn't in v1's published schema, but v1 honours it. Without it, the
    transaction may have no originator institution and stay invisible to the institution that owes the data.
- **Afterwards:**
  - The **originator institution** adds its customer's data with `POST /tx/update` while the transaction is
    `INCOMPLETE` (v1 accepts originator data only at `INCOMPLETE` or `MISSING_BENEFICIARY_DATA`). The beneficiary can
    read it back in cleartext via `/tx/info`.
  - The **beneficiary institution** then sends its beneficiary persons with `/tx/update`. A beneficiary caller
    updating an `INCOMPLETE` notify transaction with valid data **completes the notify flow**: v1 sets `SENT`, and an
    autoConfirm rule moves it to `ACK` a second later. After that, v1 refuses the originator's `/tx/update` with a
    400. **Order matters:** let the originator supply its data first, or send both sides yourself in one update.
  - An originator-caller update yields `NEW` instead, and the originator must then `/tx/approve`.
  - From `ACK`, the beneficiary accepts as usual.
- **The `txHash` is the settlement.** Funds moved before the Travel Rule exchange. No further settlement step exists
  for a notify transaction.

---

## Address registration and ownership

Registering your customers' deposit addresses is what lets Notabene route incoming transfers to you and lets
autoConfirm fire.

**Register one address:** `POST /address/registerCustomer` returns `200` with no body.

```json
{ "vaspDID": "did:ethr:0xbbbb…", "asset": "ETH", "address": "0xBeneficiaryAddress", "customerRef": "cust-123" }
```

`vaspDID`, `address` and `customerRef` are required. `asset` is a symbol; the schema also allows the CAIP-19 and
Coingecko forms.

**Look up who owns an address:**
`GET /v1/addresses/address-ownerships/{address}?vasp_did=<did>&asset=<symbol>` returns `{ owner_vasp_did, address,
address_type (HOSTED|UNHOSTED), asset, source }`, or 404 when unknown. `owner_vasp_name` is deprecated. Note the
snake_case parameters on `/v1/addresses*`.

**List or delete your address book:**

- `GET /v1/addresses?vasp_did=<did>&source=ADDRESS_GRAPH[&source_type=INTERNAL|EXTERNAL&page=&per_page=]` returns
  `{ pagination: { page, per_page, total }, metadata: { source }, data: [{ address, owner_vasp_did, customer_did,
  customer_ref }] }`.
- `DELETE /v1/addresses?vasp_did=<did>&addresses=<a1>,<a2>` (comma-joined) answers `202 { message }`.

**Bulk upload:** `GET /address/upload/{vaspDID}` returns a presigned `{ uploadUrl, uploadFields[] }`. POST your CSV to
it as a multipart form, including every upload field verbatim.

**Address query webhook (pull instead of push):**
`POST /integrations/{vaspDID}/address-query-webhook` with `{ "url": "https://you.example/addr", "params": "?k=v" }`
registers a URL Notabene calls to ask whether an address is yours; the response carries a `secret`. `params` is a
query string with a leading `?`. `DELETE` on the same path removes it. The request Notabene sends to that URL isn't
modelled in the repo.

---

## Webhooks

### Configure

One webhook URL per institution:

```
POST   /webhook            { "vaspDID": "did:ethr:0xbbbb…", "url": "https://you.example/notabene" }
GET    /webhook?vaspDID=…  → { "vaspDID", "url" }   (404 when none is configured)
DELETE /webhook            { "vaspDID": "did:ethr:0xbbbb…" }
```

v1 rejects URLs with a single-label hostname (such as `https://webhooks/…`); use a fully qualified, dotted host. The older
`POST /integrations/config/txnotification` with `{ vaspDID, callbackUrl }` (read back with
`GET /integrations/txnotification/{vaspDID}`) is **deprecated**; use `/webhook`.

### Delivery and verification

Webhooks are delivered by **Svix**. Every delivery carries the standard `svix-id`, `svix-timestamp` and
`svix-signature` headers. Verify them with a Svix library and your endpoint's signing secret before trusting the body.
Return 2xx once you have processed the delivery; any non-2xx makes Svix redeliver. Notabene keeps one Svix application
per institution, and each application's endpoint has its own signing secret. How an integrator obtains that secret
(dashboard or support) isn't shown in the repo.

### Payload

```json
{
  "message": "notification.transactionUpdated",
  "payload": {
    "transaction": {
      "id": "6f1c…-uuid",
      "transactionRef": "withdrawal-8841",
      "status": "ACK",
      "transactionAsset": "ETH",
      "transactionAmount": "1000000000000000000",
      "originatorVASPdid": "did:ethr:0xaaaa…",
      "beneficiaryVASPdid": "did:ethr:0xbbbb…",
      "transactionBlockchainInfo": { "txHash": "0x9c1d…", "destination": "0xBeneficiaryAddress" }
    }
  },
  "version": "…"
}
```

`message` is the event type. Every event carries exactly one transaction under `payload.transaction`.

| `message` | When |
| --- | --- |
| `notification.transactionUpdated` | On every status or data change after creation. The one that drives the lifecycle; branch on `payload.transaction.status`. |
| `notification.processBlockchainTransaction` | When v1 sees the originating on-chain transfer and asks you to record or process it. |
| `notification.haltBlockchainTransaction` | When a downstream signal (sanctions, risk score, manual hold) takes the transfer out of the active path. Stop processing it. |

Handling rules:

- The embedded transaction may be **PII-stripped**. Re-read `GET /tx/info?id=…` when you need the persons.
- Deliveries can **repeat** and arrive **out of order**. Make handlers idempotent: key them on `id` + `status`, and
  never move your own state backwards when a late, older status arrives.
- Your own call can trigger a webhook before the call returns. Correlate on `transactionRef` (which you set on create)
  so a webhook that beats your response still finds its record.
- A `transactionBlockchainInfo.txHash` starting with `pending-` is a **placeholder** some Notabene components seed
  before settlement. It is not an on-chain hash.

---

## Customer token for the JavaScript SDK

The v1 JS SDK widgets run in your customer's browser with a short-lived, customer-scoped token that your backend mints:

```
POST /auth/customerToken
{ "vaspDID": "did:ethr:0xaaaa…", "customerRef": "cust-123" }
```

The response is `{ "access_token": "…", "token_type": "Bearer", "scope": "…" }`. Both fields are required. Mint the
token server-side with your institution token and hand only `access_token` to the front end; never ship your client
secret to the browser.

Related: `POST /auth/customer` with `{ "vaspDID", "customerRef" }` creates a customer record and returns
`{ id, did, vaspDID, customerRef }`. `GET /auth/customer/{vaspDID}/{customerDID}` reads it back.

---

## Reading and listing transactions

| Call | Notes |
| --- | --- |
| `GET /tx/info?id=<uuid>` | One transaction. An empty `200` means you can't see it. |
| `GET /tx/list?vaspDID=<did>&…` | Paginated list (`vaspDID` is required). Returns `{ transactions: [...], paginateInfo: { totalItems, itemCount, itemsPerPage, totalPages, currentPage } }`. List items also carry `originatorRef`, `beneficiaryRef` and `transactionAssetDecimals`. |
| `GET /tx/query?txHash=&destination=&vout=` | Find transactions by on-chain coordinates; returns an array. |
| `GET /tx/listBeneficiaryOriginator?vaspDID=<did>` | The same envelope as `/tx/list`. |

Filters for `/tx/list`:

| Parameter | Values |
| --- | --- |
| `sort` | `transactionAmount:ASC\|DESC`, `updatedAt:ASC\|DESC`, `createdAt:ASC\|DESC` |
| `page`, `resultsPerPage` | Pagination. |
| `txGroup` | `INBOX` limits the list to the inbox view. |
| `txDirection` | `incoming`, `outgoing`, `internal` |
| `status` | Repeatable. |
| `transactionAsset`, `transactionType`, `transactionRef` | Exact-match filters. |
| `q` | Free-text search. |
| `start`, `end` | Dates as `YYYY-MM-DD`. |
| `jurisdictions`, `vasps` | Repeatable. |
| `wallet` | Filter by wallet address. |
| `decrypt`, `includeActions`, `csv` | Booleans. |

---

## Gotchas

1. **Amounts are integer base units.** `transactionAmount` is a string of the smallest unit: wei, satoshi, or 10^-6 for
   USDC. `"1000000000000000000"` is 1 ETH, not 10^18 ETH. Notabene's own v1↔current bridge multiplies by 10^decimals
   on the way into v1 and divides on the way out. `amountInLocalCurrency.amountInLocalCurrency`, by contrast, is a
   plain JSON number in fiat; parse it as a decimal, not a float. The client also rejects absurd exponents like
   `1e2000000000`.
2. **Assets are symbols.** Send `ETH` or `USDC`, not CAIP-19, unless you deliberately use the `{ "caip19": … }` object
   form. An asset Notabene can't price by symbol needs `customAssetPrice`. Do **not** send `customAssetPrice` for an
   asset it already prices: the `{ price, currency }` form is feature-flagged and only accepts `USD` or `EUR`, so it
   can fail the whole create.
3. **`did:ethr` institution DIDs.** v1 institution DIDs are mostly `did:ethr:0x<address>`. They have no DIDComm
   endpoint and no web-resolvable document; they are identifiers inside Notabene's directory. Look an institution up
   with `GET /tf/simple/vasps/{vaspDID}` or search with `GET /tf/simple/vasps`. When the same institution exists on
   the current platform, its DID there is a `did:web`; don't assume the two are interchangeable.
4. **Customer DIDs aren't always DIDs.** `originatorDid` and `beneficiaryDid` may hold `did:email:…` and other IRIs.
   Store them as opaque strings.
5. **Confirm ≠ accept.** `/tx/confirm` asserts that you control the destination address; `/tx/accept` is the
   compliance decision. Notabene had to remove code that confirmed on a counterparty's behalf because it made
   transfers complete with nobody on the beneficiary side acting. Never confirm an address you haven't checked.
6. **Accept needs `ACK`.** An accept against `SENT` is refused, even though the published status map lists `SENT`.
   Confirm (or let autoConfirm run), then accept, and allow a short retry for the read lag.
7. **Rules are asynchronous.** Auto-approve (`NEW → SENT`) and autoConfirm (`SENT → ACK`) lag the call that triggered
   them by up to several seconds. A beneficiary that accepts a `NEW` transaction can land it in `ACCEPTED` without
   `SENT` ever being published, so wait for `SENT` before acting.
8. **Silent success on writes you can't make.** `/tx/update` returns 200 and does nothing when your token doesn't
   cover the transaction. `/tx/info` returns an empty 200 for the same reason. Re-read to verify.
9. **Omit, don't blank.** An empty-string DID on `/tx/update` clears the stored value. An empty `txHash` on
   `/tx/notify` fails validation. Leave unset keys out of the JSON entirely.
10. **PII escrow and encryption.** v1 encrypts originator PII to the originator institution's escrow key and releases
    it to the beneficiary only after it confirms. Before that, the originator block is `{}` for the beneficiary.
    Cleartext names you post are stored encrypted (IPFS-backed); not even `/tx/list?decrypt=true` resolves every
    field. Keep your own copy of what you sent. `ivms101Encrypted`, `encrypted` and `pii` carry the encrypted and
    alternate forms; the plain `originator` and `beneficiary` blocks are the common path.
11. **Idempotency is yours to build.** `/tx/create` has no idempotency key. Set a unique `transactionRef` and, before
    retrying a create that timed out, look it up with `GET /tx/list?vaspDID=…&transactionRef=…` to avoid duplicates.
    Treat webhooks as at-least-once.
12. **Above-threshold data rules.** Each originator person needs a name and an account number, plus a national id,
    date and place of birth, or geographic address. Beneficiary persons are mandatory above threshold. Validate with
    `/tx/validate/full` first.
13. **Deposit-notify ordering.** On a notify transaction, a beneficiary `/tx/update` with full data ends the window in
    which the originator can add its data. See [Deposit first](#deposit-first-txnotify).
14. **Two pagination shapes.** `/tx/list` uses `paginateInfo { totalItems, … }`, while `/v1/addresses` uses
    `pagination { page, per_page, total }`. Query parameter casing also differs (`vaspDID` vs `vasp_did`).

---

## Other endpoints

v1 has more endpoints that are out of scope here. The rest of the surface also covers:

- Institution directory and admin: `/tf/simple/vasps…` for lookups, plus institution creation and updates, settings
  and deletion under `/tf/vasps/…`.
- Users and credentials: `/auth/vaspUsers`, `/auth/addusertovasp`, and `/auth/credentials/{vaspDID}` (create,
  rotate, delete and read client credentials).
- Transaction rules: `/rules/get`, `/rules/set/direction`, `/rules/delete`.
- Documents and DDQ: `/tf/vasps/{vaspDID}/documents/…`.
- Screening-provider credentials and lookups: `/integrations/{chainalysis,trm,elliptic,…}`.
- Protocol agents and gateway callbacks: `/protocol-agents`, `/protocol-gateways/callbacks/…`.
- Jurisdictions: `/jurisdictions`.
- Payments (Flow): signalled by the `isPayout` flags.

---

## Unverified facts

These facts could not be confirmed from the repository this guide was built from:

- **Token URL.** The production v1 token URL lives in a secret. `https://auth.notabene.id/oauth/token` is inferred from
  the production Auth0 issuer.
- **Integrator audience.** The repo shows only the privileged node-tier audience (`https://node.api.notabene.id`) and
  a test harness that uses the API base URL as the audience. The audience a regular institution should request is
  presumed to be the API base URL; confirm it in your onboarding material.
- **Webhook signing secret.** How an institution obtains its Svix signing secret, and the value of the envelope's
  `version` field, aren't shown.
- **Unexercised endpoints.** The exact semantics of `/tx/receive`, `/tx/updateTR`, `NOT_READY`,
  `WAITING_FOR_INFORMATION` and the `processBlockchainTransaction` / `haltBlockchainTransaction` events are described
  from type definitions and code comments only. Nothing in the repo exercises them end to end.
- **US region.** No separate US base URL for the v1 API appears in the config.
