# Getting started with the v1 (legacy) Travel Rule API

v1 is Notabene's original REST API for Travel Rule transfers between institutions. An institution (`vaspDID`) drives
transactions through verb endpoints (`/tx/create`, `/tx/confirm`, `/tx/accept`, …) and receives Svix-signed webhooks
carrying the whole transaction. Names and enums are verbatim from Notabene's typed Go v1 client. Gotchas live in one
place: [Caveats](caveats.md).

## Quick start

1. Get an OAuth client-credentials token; send `Authorization: Bearer <access_token>` on every call.
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

| Environment | Base URL |
| --- | --- |
| Production | `https://api.notabene.id` |
| QA / staging (EU) | `https://api-qa.eu.notabene.dev` |
| "test" (v1's sandbox) | `https://api.notabene.dev` |

Paths hang off the base URL (`https://api.notabene.id/tx/create`); the `/v1/addresses…` endpoints carry `/v1`,
other address calls (`/address/registerCustomer`, `/address/upload/{vaspDID}`) don't. Every endpoint needs a bearer
token from the client-credentials grant:

```
POST <token URL>
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=<clientId>&client_secret=<clientSecret>&audience=<audience>
```

Refresh before expiry. Credentials are minted per institution (`POST /auth/credentials/{vaspDID}`, admin, returns
`{ "clientId", "clientSecret" }`) and the JWT is scoped to that `vaspDID`. Token URL and audience are not confirmed:
see [Caveats § Unverified facts](caveats.md#unverified-facts).

## Errors and response quirks

```json
{ "err": { "name": "NotabeneTransactionError", "code": 400, "message": "…", "stack": "…" } }
```

`code` usually echoes the HTTP status. 400-class: validation or illegal transition (validation can list fields, e.g.
`["beneficiaryNameMissing","beneficiaryAccountNumberMissing"]`). 401: bad/expired token or not a member of the scoped
institution. 404: real not-found on some endpoints (`GET /webhook` with none set, address ownership lookups).

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
| `transactionAmount` | String, **integer in the smallest unit**: `"1000000000000000000"` = 1 ETH. |
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

| Status | Meaning |
| --- | --- |
| `SAVED` | Stored, not sent (below-threshold creates park here; no `SENT` webhook). |
| `NEW` | Awaiting the originator's approval. |
| `MISSING_BENEFICIARY_DATA` | Created without required beneficiary data; still approvable, accepts `/tx/update` data. |
| `WAITING_FOR_INFORMATION` | Waiting for party data. Not exercised; for `/tx/update` it is originator-only, unlike `INCOMPLETE`. |
| `INCOMPLETE` | `/tx/notify` deposit without originator data. |
| `SENT` | Delivered; awaiting the beneficiary's confirm, reject or redirect. |
| `ACK` | Beneficiary confirmed it controls the destination; awaiting accept or decline. |
| `ACCEPTED` | Terminal success. |
| `REJECTED`, `DECLINED`, `CANCELLED` | Terminal: rejected at `SENT`, declined after review, cancelled by originator. |
| `NOT_READY` | Beneficiary called `/tx/notReady`. |

Happy paths: `NEW → SENT → ACK → ACCEPTED`; deposit notification, originator data first,
`INCOMPLETE → NEW → (approve) SENT → ACK → ACCEPTED` ([both orders](receive.md#deposit-first-txnotify)). Log and
ignore unknown statuses (e.g. `MERGED`).

## Who may call which verb

| Verb | Caller | Allowed from | Result |
| --- | --- | --- | --- |
| `POST /tx/approve?id=` | originator | `NEW`, `MISSING_BENEFICIARY_DATA` | `SENT` |
| `POST /tx/confirm?id=` | beneficiary | `SENT` | `ACK` |
| `POST /tx/reject` | beneficiary | `SENT` | `REJECTED` |
| `POST /tx/redirect?id=&to=` | beneficiary | `SENT` (one source also lists `NEW`, `MISSING_BENEFICIARY_DATA`, `WAITING_FOR_INFORMATION`; unconfirmed) | Status unchanged; `beneficiaryVASPdid` becomes `to` |
| `POST /tx/accept?id=` | beneficiary | `ACK` (map also lists `SENT`, `SAVED`, `INCOMPLETE`; `SENT` is **refused** in practice) | `ACCEPTED` |
| `POST /tx/decline` | beneficiary | `ACK`, `SENT`, `SAVED`, `INCOMPLETE` | `DECLINED` |
| `POST /tx/cancel` | originator | non-terminal; can be refused at `SENT` / `ACK` (then only the beneficiary's decline ends it) | `CANCELLED` |
| `POST /tx/update` (`txHash`) | either in `INCOMPLETE`, `MISSING_BENEFICIARY_DATA`, `SAVED`; originator only in other non-terminal statuses (beneficiary limited to `comment`) | any, incl. `ACCEPTED` | Records the hash |
| `POST /tx/update` (party data) | depends | `INCOMPLETE`, `MISSING_BENEFICIARY_DATA` (also `SAVED`) | Re-validated; may move status ([deposit flow](receive.md#deposit-first-txnotify)) |

Institution **rules** act asynchronously: an outgoing *approve* rule advances `NEW → SENT` during `/tx/create`; a
beneficiary *autoConfirm* rule advances `SENT → ACK` seconds later when the destination is in its address book. Wait
for the webhook or poll.

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
| `GET /tx/info?id=<uuid>` | One transaction; empty `200` = not visible to you. |
| `GET /tx/list?vaspDID=<did>&…` | `{ transactions: [...], paginateInfo: { totalItems, itemCount, itemsPerPage, totalPages, currentPage } }`. Items add `originatorRef`, `beneficiaryRef`, `transactionAssetDecimals`. |
| `GET /tx/query?txHash=&destination=&vout=` | By on-chain coordinates; returns an array. |
| `GET /tx/listBeneficiaryOriginator?vaspDID=<did>` | Same envelope as `/tx/list`. |

`/tx/list` filters: `sort` (`transactionAmount|updatedAt|createdAt` + `:ASC|DESC`), `page`, `resultsPerPage`,
`txGroup=INBOX`, `txDirection` (`incoming|outgoing|internal`), `status`, `transactionAsset`, `transactionType`,
`transactionRef`, `q`, `start`/`end` (`YYYY-MM-DD`), `jurisdictions`, `vasps`, `wallet`, `decrypt`, `includeActions`,
`csv`. `status`, `jurisdictions`, `vasps` repeat.

## Other endpoints exist

Out of scope: directory and admin (`/tf/simple/vasps…`, `/tf/vasps/…`), users and credentials (`/auth/vaspUsers`,
`/auth/credentials/{vaspDID}`), rules (`/rules/…`), documents (`/tf/vasps/{vaspDID}/documents/…`), screening
(`/integrations/{chainalysis,trm,elliptic,…}`), protocol agents and gateways, `/jurisdictions`, payments (`isPayout`).
