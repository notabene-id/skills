# Getting started with the v1 (legacy) Travel Rule API

v1 is Notabene's original REST API for Travel Rule transfers between institutions. An institution (`vaspDID`) drives
transactions through verb endpoints (`/tx/create`, `/tx/confirm`, `/tx/accept`, …) and receives Svix-signed webhooks
carrying the whole transaction. Names and enums are verbatim from Notabene's typed Go v1 client. Gotchas live in one
place: [Caveats](caveats.md).

## Contents

- [Quick start](#quick-start)
- [Recognising v1](#recognising-v1)
- [Base URLs and authentication](#base-urls-and-authentication)
- [Errors and response quirks](#errors-and-response-quirks)
- [The transaction object](#the-transaction-object)
- [Statuses](#statuses)
- [Who may call which verb](#who-may-call-which-verb)
- [Customer token for the JavaScript SDK](#customer-token-for-the-javascript-sdk)
- [Reading and listing transactions](#reading-and-listing-transactions)
- [Other endpoints exist](#other-endpoints-exist)

## Quick start

1. Get an OAuth client-credentials token from `https://auth.notabene.id/oauth/token` (audience
   `https://api.notabene.id`, test `https://api.notabene.dev`); send `Authorization: Bearer <access_token>` on every
   call.
2. Configure and verify webhooks ([Webhooks](webhooks.md)).
3. Sending: [Send](send.md). Receiving: [Receive](receive.md).
4. Read a transaction with `GET /tx/info?id=<uuid>`; an empty `200` means "not found / not yours".

## Recognising v1

- `vaspDID` (mostly `did:ethr:0x<40 hex>`, some `did:web`) as a query parameter or body field on almost every call;
  the current API uses `entityDID` (`did:web`) in resource paths under `/entities/{entityDID}/…`.
- Verb paths: `/tx/create`, `/tx/info?id=`, `/tx/confirm?id=`, `/tx/notify`, `/webhook`.
- Fields `originatorVASPdid` / `beneficiaryVASPdid` (lowercase `did`); IVMS in `originatorPersons[]` /
  `beneficiaryPersons[]` (top-level `originator` / `beneficiary` in requests, under `ivms101` in responses);
  statuses `NEW`, `SENT`, `ACK`, `ACCEPTED`, …
- Webhook bodies `{ "message": "notification.transactionUpdated", "payload": { "transaction" }, … }`; error bodies
  `{ "err": { … } }`.

## Base URLs and authentication

| Environment | Base URL | Token `audience` | Dashboard |
| --- | --- | --- | --- |
| Production | `https://api.notabene.id` | `https://api.notabene.id` | `https://app.notabene.id` |
| "test" (v1's sandbox) | `https://api.notabene.dev` | `https://api.notabene.dev` | `https://app.notabene.dev` |
| QA / staging (EU, Notabene-internal) | `https://api-qa.eu.notabene.dev` | | |

Paths hang off the base URL (`https://api.notabene.id/tx/create`); the `/v1/addresses…` endpoints carry `/v1`,
other address calls (`/address/registerCustomer`, `/address/upload/{vaspDID}`) don't. Every endpoint needs a bearer
token from the client-credentials grant:

```
POST https://auth.notabene.id/oauth/token
Content-Type: application/json

{ "grant_type": "client_credentials", "client_id": "<clientId>", "client_secret": "<clientSecret>",
  "audience": "https://api.notabene.id" }
```

Use audience `https://api.notabene.dev` for the test host; both environments share the issuer
`https://auth.notabene.id/` (RS256, keys at `https://auth.notabene.id/.well-known/jwks.json`). The response is a
standard OAuth2 token response (`access_token`, `expires_in`, `token_type`): refresh before `expires_in` runs out
rather than hard-coding a lifetime. The API host's own `/oauth/token` is a password grant for the dashboard's login,
not for integrations.

Credentials are minted per institution and the JWT is scoped to that `vaspDID`: an institution admin creates them in the
dashboard or with `POST /auth/credentials/{vaspDID}` (body `{ "limited"?, "delegateToGateway"? }`, returns
`{ "clientId", "clientSecret" }`); `PUT` / `DELETE /auth/credentials/{vaspDID}?clientid=…` rotate and delete them.

## Errors and response quirks

```json
{ "err": { "name": "NotabeneTransactionError", "code": 400, "message": "…", "stack": "…" } }
```

`code` usually echoes the HTTP status. 400-class: validation or illegal transition (validation can list fields, e.g.
`["beneficiaryNameMissing","beneficiaryAccountNumberMissing"]`). 401: bad/expired token or not a member of the scoped
institution. 404: real not-found on some endpoints (`GET /webhook` with none set, address ownership lookups).

| Situation | Status, `err.name` |
| --- | --- |
| Missing or invalid token | `401` `UnauthorizedError` |
| Caller may not act for this institution | `400`/`403` `UserDIDNotAdminforVASPError` or a permission error |
| Validation failed on create | `400` `NotabeneTransactionValidationError` (`message` lists the missing fields) |
| Unsupported asset | `400` `NotabeneCryptoAssetNotAvailable` |
| No subscription, plan limits | `400` `NotabeneTransactionNoActiveSubscriptionError`, `NotabenePaymentNoCustomerError`, `NotabeneTransactionSunriseLimitReachedError` |
| Verb not allowed in the current status | `400` `NotabeneTransactionInvalidStatusError` |
| Unknown transfer id on update | `NotabeneTransactionMissingError` |
| `/tx/list` date range outside the plan's audit-trail window | `400` `{ "error": "Invalid date range. Please check your plan limits." }` |

On a status error, re-read the transaction and act on the status it is actually in; don't retry the same verb.

- `GET /tx/info` returns `200` with an empty body (or `{}`) for a transaction your token can't see.
- `POST /tx/update` returns `200` and does nothing when your token can't mutate the transaction.
- `POST /tx/bulk` returns `200` + array of transactions, or `202` + array of id strings (queued).

## The transaction object

Returned by `/tx/create`, `/tx/info`, every verb, and every webhook.

| Field | Notes |
| --- | --- |
| `id` | Notabene's UUID; used on every later call. |
| `transactionRef` | The originator's reference. Redacted from webhooks delivered to the beneficiary side: originators may correlate on it, beneficiaries on `id`. |
| `status`, `transactionType` | See [statuses](#statuses). Type: `BELOW_THRESHOLD`, `NON_CUSTODIAL`, `TRAVELRULE`, `UNKNOWN`. |
| `transactionAsset` | **Symbol** (`ETH`, `USDC`). Create also accepts `{ "caip19" }` or `{ "coingeckoId", "network" }`. |
| `transactionAmount` | String, **integer in the smallest unit** (`^0*[1-9][0-9]*$`): `"1000000000000000000"` = 1 ETH. |
| `amountInLocalCurrency`, `beneficiaryAmountInLocalCurrency` | `{ "amountInLocalCurrency": <JSON number>, "currency": "USD" }` per jurisdiction. |
| `originatorDid`, `beneficiaryDid` | Customer IRIs (lowercase `id`); may be `did:email:…`. |
| `originatorVASPdid`, `beneficiaryVASPdid` | The two institutions. |
| `transactionBlockchainInfo` | `{ "txHash", "origin", "destination" }`; `destination` required on create. |
| `ivms101` | IVMS 101 envelope (below): the persons live in `ivms101.originator` / `ivms101.beneficiary`. |
| `originatorProof`, `beneficiaryProof` | Ownership proof (below). |
| `actions` | Audit log `{ "hash", "issuerDID", "action", "actionData" }`. |

Other fields: `chargedQuantity`; `pii` (mirror of `ivms101`, same shape; read whichever is present) and
`ivms101Encrypted`; `protocol` (`TRLight`, `TRP`, `OpenVASP`) and
`protocolData` (`{ "trLightData": { "notificationEmails" } }`); `comments`; `isPayout` (payments, ignore).

### IVMS 101 envelope

A response has **no top-level `originator` / `beneficiary`**: read the persons from `ivms101` (or its `pii` mirror).
Request bodies (`/tx/create`, `/tx/update`) take `originator` / `beneficiary` at the top level instead. The envelope:

```json
{
  "originator": { "originatorPersons": [ { "naturalPerson": { … } } ], "accountNumber": ["<account id>"] },
  "beneficiary": { "beneficiaryPersons": [ { "legalPerson": { … } } ], "accountNumber": ["<account id>"] },
  "originatingVASP": { "originatingVASP": { "legalPerson": { … } } },
  "beneficiaryVASP": { "beneficiaryVASP": { "legalPerson": { … } } },
  "transferPath": { "transferPath": [ { "intermediaryVASP": { … }, "sequence": 1 } ] },
  "payloadMetadata": { "transliterationMethod": ["othr"] }
}
```

- A person is `{ "naturalPerson" }` **or** `{ "legalPerson" }`, never both. Full example: [Send](send.md#2-create).
- `naturalPerson`: `name[]` (`nameIdentifier[]`, `localNameIdentifier[]`, `phoneticNameIdentifier[]` of
  `{ primaryIdentifier, secondaryIdentifier, nameIdentifierType }`), `geographicAddress[]`, `nationalIdentification`,
  `customerIdentification`, `dateAndPlaceOfBirth { dateOfBirth (YYYY-MM-DD), placeOfBirth }`, `countryOfResidence`.
- `legalPerson`: `name { nameIdentifier[] of { legalPersonName, legalPersonNameIdentifierType } }`,
  `geographicAddress[]`, `customerNumber`, `nationalIdentification`, `dateOfRegistration`, `countryOfRegistration`.
- Address: `addressType` (`HOME|BIZZ|GEOG`), `streetName`, `buildingNumber`, `buildingName`, `postCode`, `townName`,
  `countrySubDivision`, `addressLine[]`, `country` (ISO 3166 alpha-2).
- The natural-person name type key is `nameIdentifierType`, not IVMS's (and v2.1's)
  `naturalPersonNameIdentifierType`.
- Codes: `nameIdentifierType` `LEGL|ALIA|BIRT|MAID|MISC`; `legalPersonNameIdentifierType` `LEGL|SHRT|TRAD`;
  `nationalIdentifierType` `ARNU|CCPT|RAID|DRLC|FIIN|TXID|SOCS|IDCD|LEIX|MISC`.
- `primaryIdentifier` is the **last name**; `secondaryIdentifier` the first names.
- `accountNumber` is the institution's account id for the customer, not the on-chain address (often reused anyway).

Ownership proofs, selected by `type` (relevant for self-hosted destinations; validation can report
`beneficiaryOwnershipProofMissing`):

| `type` | Fields |
| --- | --- |
| `eip-191`, `eip-712`, `bip-137` | `proof` (signature), `attestation` (signed message), `address`, `status` |
| `screenshot` | `url` |
| `self-declaration` | `attestation`, `did`, `address`, `confirmed` (bool), `status` |
| `microtransfer` | `proof`, `did`, `address`, `chain`, `destination`, `amountSubunits`, `status` |

## Statuses

The published enum has the 12 values below.

| Status | Meaning |
| --- | --- |
| `SAVED` | Stored, not sent to a counterparty: below threshold, a self-hosted wallet, or internal. No `SENT` webhook follows; the originator broadcasts and records the hash. |
| `NEW` | Awaiting the originator's approval (`/tx/approve` or an outgoing rule). |
| `MISSING_BENEFICIARY_DATA` | Created without required beneficiary data; still approvable, accepts `/tx/update` data. |
| `WAITING_FOR_INFORMATION` | Approved, but the beneficiary institution is unclaimed on Notabene and has no notification email: add `notificationEmail` with `/tx/update` (originator-only in this status, unlike `INCOMPLETE`). |
| `INCOMPLETE` | `/tx/notify` deposit without originator data. |
| `SENT` | Delivered; awaiting the beneficiary's confirm, reject or redirect. |
| `ACK` | Beneficiary confirmed it controls the destination; awaiting accept or decline. |
| `ACCEPTED` | Terminal success. |
| `REJECTED`, `DECLINED`, `CANCELLED` | Terminal: rejected at `SENT`, declined after review, cancelled by originator. |
| `NOT_READY` | Beneficiary called `/tx/notReady` at `SENT`: not ready to respond. |

Happy paths: `NEW → SENT → ACK → ACCEPTED`; deposit notification, originator data first,
`INCOMPLETE → NEW → (approve) SENT → ACK → ACCEPTED` ([both orders](receive.md#deposit-first-txnotify)).

The service also stores `MERGED` (a beneficiary's `INCOMPLETE` transaction folded into the originator's when the
originator reports the same hash) and `WAITING_FOR_TR` (set by the internal `/tx/receive`); handle them, and any other
unknown status, as valid but not actionable. `/tx/list` also accepts the filter-only values
`MISSING_DATA_REQUESTED_LT_5_DAYS`, `MISSING_DATA_REQUESTED_GT_5_DAYS`, `ORIGINATOR_UNREACHABLE`, which are never
stored.

## Who may call which verb

| Verb | Caller | Allowed from | Result |
| --- | --- | --- | --- |
| `POST /tx/approve?id=` | originator | `NEW`, `MISSING_BENEFICIARY_DATA` (the v1 service also allows `INCOMPLETE` on Travel Rule transfers) | `SENT`; `WAITING_FOR_INFORMATION` if the beneficiary institution is unclaimed with no notification email; `SAVED` for below-threshold, self-hosted and internal transfers |
| `POST /tx/confirm?id=` | beneficiary | `SENT` | `ACK` |
| `POST /tx/reject` | beneficiary | `SENT` | `REJECTED` |
| `POST /tx/redirect?id=&to=` | beneficiary | `SENT` (one source also lists `NEW`, `MISSING_BENEFICIARY_DATA`, `WAITING_FOR_INFORMATION`; unconfirmed) | Status unchanged; `beneficiaryVASPdid` becomes `to` |
| `POST /tx/accept?id=` | beneficiary | `ACK` (map also lists `SENT`, `SAVED`, `INCOMPLETE`; `SENT` is **refused** in practice) | `ACCEPTED` |
| `POST /tx/decline` | beneficiary | `ACK`, `SENT`, `SAVED`, `INCOMPLETE` | `DECLINED` |
| `POST /tx/cancel` | originator | `NEW`, `MISSING_BENEFICIARY_DATA`, `WAITING_FOR_INFORMATION`, `SENT` (and `INCOMPLETE` on Travel Rule transfers); refused at `ACK`, and seen refused at `SENT` on deposit-notify transactions (then only the beneficiary's decline ends it) | `CANCELLED` |
| `POST /tx/update` (`txHash`, `origin`, `destination`) | either in `INCOMPLETE`, `MISSING_BENEFICIARY_DATA`, `SAVED`; originator only in other non-terminal statuses (beneficiary limited to `comment`) | any, incl. `ACCEPTED` | Records the hash; may merge a matching `INCOMPLETE` deposit notification ([Send § 5](send.md#5-record-the-on-chain-hash)) |
| `POST /tx/update` (party data) | depends | `INCOMPLETE`, `MISSING_BENEFICIARY_DATA` (also `SAVED`) | Re-validated; may move status ([deposit flow](receive.md#deposit-first-txnotify)) |

Institution **rules** act asynchronously. Outgoing rules (configured in the dashboard) run during `/tx/create`: a rule
can `APPROVE` (advancing `NEW → SENT`), `REJECT` (which cancels the transfer), `SEND_TO_INBOX` (leave it for a human)
or `NOT_READY`; if none fires, the transfer stays `NEW`. Internal transfers (same originator and beneficiary
institution) are approved automatically. After approval Notabene tries to *auto-confirm*: if the destination is in the
beneficiary institution's address book, or its address-query webhook answers `owns: true`, the transfer moves
`SENT → ACK` seconds later; `owns: false` auto-rejects it
([Receive § Address registration](receive.md#address-registration-and-ownership)). Incoming rules can accept or
decline automatically. Wait for the webhook or poll.

## Customer token for the JavaScript SDK

The v1 JS SDK widgets use a short-lived, customer-scoped token your backend mints with its institution token:

```
POST /auth/customerToken  { "vaspDID": "did:ethr:0xaaaa…", "customerRef": "cust-123" }
→ { "access_token": "…", "token_type": "Bearer", "scope": "…" }
```

Both request fields are required. Give only `access_token` to the front end; never ship the client secret.
(`POST /auth/customer` with the same body creates a customer record `{ id, did, vaspDID, customerRef }`.)

## Reading and listing transactions

| Call | Notes |
| --- | --- |
| `GET /tx/info?id=<uuid>[&vaspDID=<did>]` | One transaction; empty `200` = not visible to you. Optional `vaspDID` picks which of your institutions you act as. |
| `GET /tx/list?vaspDID=<did>&…` | `{ transactions: [...], paginateInfo: { totalItems, itemCount, itemsPerPage, totalPages, currentPage } }`. Items add `originatorRef`, `beneficiaryRef`, `transactionAssetDecimals`. |
| `GET /tx/query?txHash=&destination=&vout=` | By on-chain coordinates; returns an array. |
| `GET /tx/listBeneficiaryOriginator?vaspDID=<did>` | Same envelope as `/tx/list`. |

`/tx/list` filters: `sort` (`transactionAmount|updatedAt|createdAt` + `:ASC|DESC`), `page` (from 0), `resultsPerPage`
(max 100),
`txGroup=INBOX`, `txDirection` (`incoming|outgoing|internal`), `status`, `transactionAsset`, `transactionType`,
`transactionRef`, `q`, `start`/`end` (`YYYY-MM-DD`), `jurisdictions`, `vasps`, `wallet`, `decrypt`, `includeActions`,
`csv`. `status`, `jurisdictions`, `vasps` repeat.

## Other endpoints exist

Out of scope: directory and admin (`/tf/simple/vasps…`, `/tf/vasps/…`), users and credentials (`/auth/vaspUsers`,
`/auth/credentials/{vaspDID}`), rules (`/rules/…`), documents (`/tf/vasps/{vaspDID}/documents/…`), screening
(`/integrations/{chainalysis,trm,elliptic,…}`), protocol agents and gateways, `/jurisdictions`, payments (`isPayout`).
Institution settings (`PUT /tf/vasps/{vaspDID}/settings`, `GET /tf/vasps/{vaspDID}/getSettings`) matter for two
keys: `statusToProcessBlockchain` ([Send § 4](send.md#4-wait-for-the-safe-to-send-signal)) and `autoconfirm`
([Receive § Address registration](receive.md#address-registration-and-ownership)).
