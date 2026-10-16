# Send a transfer (originator, v1)

As the originator institution you create a transaction carrying both customers' IVMS 101 data, approve it, wait for the
safe-to-send signal (normally after the beneficiary institution confirms the address and accepts), and only then
broadcast on-chain and record the hash.
Statuses and the verb table are in [Getting started](getting-started.md); pitfalls in [Caveats](caveats.md).

## Contents

- [Quick start](#quick-start)
- [1. Validate (optional)](#1-validate-optional)
- [2. Create](#2-create)
- [3. Approve](#3-approve)
- [4. Wait for the safe-to-send signal](#4-wait-for-the-safe-to-send-signal)
- [5. Record the on-chain hash](#5-record-the-on-chain-hash)
- [6. Cancel](#6-cancel)
- [/tx/updateTR](#txupdatetr)

## Quick start

1. `POST /tx/create` with a unique `transactionRef`, the amount in base units, and
   `transactionBlockchainInfo.destination`.
2. If the response is `NEW`, `POST /tx/approve?id=<id>` → `SENT` (skip if an auto-approve rule already did it).
3. Wait for the webhook `notification.processBlockchainTransaction` (fired at your `statusToProcessBlockchain`
   setting, usually `ACCEPTED`), or, without that setting, for `status: "ACCEPTED"`. Stop on
   `notification.haltBlockchainTransaction` or `REJECTED` / `DECLINED` / `CANCELLED`. If the create returns `SAVED`
   instead, nothing is sent to a counterparty (below threshold, a self-hosted wallet, or an internal transfer) and
   nothing will arrive to wait for: go straight to step 4.
4. Broadcast, then `POST /tx/update` with `{ "id", "txHash" }`.

## 1. Validate (optional)

`POST /tx/validate/full` dry-runs a create: the `/tx/create` body plus `validatePartyFields` (`BENEFICIARY`,
`ORIGINATOR` or `ALL`).

```json
{
  "transactionAsset": "ETH",
  "transactionAmount": "1000000000000000000",
  "originatorVASPdid": "did:ethr:0xaaaa…",
  "transactionBlockchainInfo": { "destination": "0xBeneficiaryAddress" },
  "validatePartyFields": "BENEFICIARY"
}
```

Response `{ isValid, type, beneficiaryAddressType, addressSource, beneficiaryVASPdid, beneficiaryVASPname, errors[],
warnings[], beneficiary, id }`: `type` is the `transactionType` (does the Travel Rule apply);
`beneficiaryAddressType` is `HOSTED|UNHOSTED|UNKNOWN`; `addressSource` is `ADDRESS_BOOK|ADDRESS_GRAPH|ADDRESS_HASH|
CHAINALYSIS|ELLIPTIC|UNKNOWN`; `beneficiaryVASPdid` is set when the destination resolved to an institution.

`POST /tx/validate` is **deprecated**: a flat body (`destination`, `beneficiaryName`, …, required bool
`originatorEqualsBeneficiary`) whose `errors` are `beneficiaryNameMissing`, `beneficiaryAccountNumberMissing`,
`beneficiaryAddressMissing`, `beneficiaryOwnershipProofMissing`.

## 2. Create

`POST /tx/create`:

```json
{
  "transactionRef": "withdrawal-8841",
  "transactionAsset": "ETH",
  "transactionAmount": "1000000000000000000",
  "originatorVASPdid": "did:ethr:0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "beneficiaryVASPdid": "did:ethr:0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
  "transactionBlockchainInfo": { "origin": "0xOriginAddress", "destination": "0xBeneficiaryAddress" },
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

Required: `transactionAsset`, `transactionAmount`, `originatorVASPdid`, `transactionBlockchainInfo.destination`.
Optional:

| Field | Purpose |
| --- | --- |
| `beneficiaryVASPdid` | Omit it, or send `beneficiaryVASPname` / `beneficiaryVASPwebsite`, when the counterparty is unknown. |
| `originatorDid`, `beneficiaryDid`, `originatorRef`, `beneficiaryRef` | Your customer identifiers. |
| `transactionAssetDecimals`, `network` | Asset precision and network. |
| `customAssetPrice` | oneOf `{ "priceUSD": 2.5, "decimals": 18 }` or `{ "price", "currency" }` (feature-flagged: use `priceUSD`); only for assets Notabene can't price ([Caveats](caveats.md)). |
| `originatorProof`, `beneficiaryProof` | Ownership proofs. |
| `pii` + `pii_url`, `encrypted` | Instead of cleartext `originator` / `beneficiary` (which Notabene encrypts itself): pre-encrypted IVMS-shaped `pii` **together with** `pii_url`, the PII service that can decrypt it (one without the other fails with `Missing pii and/or pii_url, both should be provided.`); or an opaque encrypted string, stored as `ivms101Encrypted` and passed through (required for `targetProtocol: "GTR"` when Notabene's GTR validation is on). |
| `protocol`, `targetProtocol`, `notificationEmail` | `targetProtocol: "GTR"` routes to GTR; `notificationEmail` for TRLight. |
| `skipBeneficiaryDataValidation` | `true` lets a create with missing beneficiary data through: it lands in `MISSING_BENEFICIARY_DATA` with `transactionType` `UNKNOWN` instead of failing. |
| `travelRuleBehavior` | `true` also applies the beneficiary jurisdiction's threshold when deciding whether the Travel Rule applies. |
| `onBehalfOfMyself` | The customer sends to their own account at another institution. |
| `blockchainTransactionTimestamp` | When the transfer happened on-chain. |

Validation observed against the real v1 stack:

- **Above threshold**, beneficiary persons are mandatory: without them `400
  ["beneficiaryNameMissing","beneficiaryAccountNumberMissing"]`; a `beneficiary` without `beneficiaryPersons` fails
  the schema.
- **Every** originator person above threshold needs a name, an account number, and a national id, date and place of
  birth, or geographic address; a name alone yields `originatorGeographicAddressMissing` or
  `originatorDateAndPlaceOfBirthMissing`.
- **Below threshold**, to a **self-hosted wallet**, or **internal** (both customers yours), it stays at `SAVED` and is
  never sent: no `SENT`/`ACCEPTED` follows. Broadcast, then record the hash (see the quick start).
- Party DIDs must match `^did:[a-zA-Z0-9]*:.*$`.

The response is the transaction at `NEW`, or `SENT` if an auto-approve rule fired (internal transfers are approved
automatically). `/tx/create` is idempotent on `transactionRef`: when a transaction with the same `transactionRef`
already exists for this `originatorVASPdid`, it returns that one instead of creating another, so always send a
persisted, unique `transactionRef` ([Caveats](caveats.md)). You can include `transactionBlockchainInfo.txHash` on
create to record a transfer after the fact.

## 3. Approve

At `NEW` or `MISSING_BENEFICIARY_DATA`: `POST /tx/approve?id=<id>` (no body). The result depends on the type:

- Travel Rule transfer: `SENT` when the beneficiary institution is claimed on Notabene or has a notification email;
  otherwise `WAITING_FOR_INFORMATION` (add `notificationEmail` with `/tx/update`).
- Below threshold, self-hosted (`NON_CUSTODIAL`) or internal: `SAVED`, nothing to exchange.

After approval Notabene tries to auto-confirm the destination for the beneficiary institution
([Getting started](getting-started.md#who-may-call-which-verb)).

## 4. Wait for the safe-to-send signal

v1 has no single "authorized" status. What is safe depends on the institution setting `statusToProcessBlockchain`
(`CREATED`, `SENT`, `ACK`, `ACCEPTED` or `null`; usually `ACCEPTED`):

```
PUT /tf/vasps/{vaspDID}/settings     { "statusToProcessBlockchain": "ACCEPTED" }
GET /tf/vasps/{vaspDID}/getSettings
```

When it is set and you have registered a [webhook](webhooks.md), Notabene sends you (as originator)
`notification.processBlockchainTransaction` when the transfer reaches:

- the configured status;
- `SAVED` (not a Travel Rule transfer), unless the setting is `CREATED`;
- `NOT_READY` (the beneficiary is not ready to respond), when the setting is `ACK` or `ACCEPTED`;
- creation itself, when the setting is `CREATED`.

It sends `notification.haltBlockchainTransaction` when the transfer becomes `CANCELLED`, `DECLINED` or `REJECTED`
(not when the setting is `CREATED`: the go-ahead was already given at creation). **Broadcast on
`notification.processBlockchainTransaction`, never after a halt.**

Without the setting there is no go-ahead event: decide from `notification.transactionUpdated` or
`GET /tx/info?id=…`. `SENT`: address not yet confirmed (a changed `beneficiaryVASPdid` means it was redirected;
re-evaluate). `ACK`: confirmed, decision pending; it only proves the address belongs to the beneficiary institution.
`ACCEPTED`: go. `REJECTED` / `DECLINED`: do not broadcast. **Never broadcast before `ACCEPTED`** on a transfer that was
sent: the API doesn't block it, but Notabene's own bridge settles only then. A transfer that stays `SAVED` was never
sent, so there is no decision to wait for.

## 5. Record the on-chain hash

After broadcasting, `POST /tx/update` as the originator (`txHash`, `origin` and `destination` can be updated in any
status, including after `ACCEPTED`; the beneficiary may set `txHash` only in `INCOMPLETE`, `MISSING_BENEFICIARY_DATA`,
`SAVED`):

```json
{ "id": "6f1c…-uuid", "txHash": "0x9c1d…" }
```

If the beneficiary institution already reported the same hash and destination with `/tx/notify` (an `INCOMPLETE`
transaction), Notabene merges it into yours and theirs becomes `MERGED`.

`/tx/update` also takes `destination`, `origin`, `originatorVASPdid`, `beneficiaryVASPdid`, `beneficiaryVASPname`,
`originatorDid`, `beneficiaryDid`, `originatorRef`, `beneficiaryRef`, `originator`, `beneficiary` (top-level in the
request, unlike the `ivms101` response), `originatorProof`, `beneficiaryProof`, `pii`, `isNonCustodial`, `comment`,
`originatorNameHash`, `beneficiaryNameHash`, `skipBeneficiaryDataValidation`, `notificationEmail`, `protocol`,
`targetProtocol`, `onBehalfOfMyself`, `blockchainTransactionTimestamp`. Party data is accepted only at `INCOMPLETE` /
`MISSING_BENEFICIARY_DATA`. `originatorVASPdid` swaps the originator institution (refused once `ACCEPTED`,
`DECLINED` or `REJECTED`). Omit keys you don't mean to change (`""` clears a field), and re-read to verify (it can
return `200` without effect).

## 6. Cancel

`POST /tx/cancel`, with optional `reason` and optional snake_case `vasp_did`:

```json
{ "id": "6f1c…-uuid", "vasp_did": "did:ethr:0xaaaa…", "reason": "customer cancelled withdrawal" }
```

Allowed from `NEW`, `MISSING_BENEFICIARY_DATA`, `WAITING_FOR_INFORMATION` and `SENT` (and `INCOMPLETE` on Travel Rule
transfers). Cancel before the beneficiary acts: it is refused at `ACK`, and has been seen refused at `SENT` on
deposit-notify transactions, where only the beneficiary can still end the transaction (`/tx/decline`); it is illegal
after `ACCEPTED`. Re-read the status first.

## `/tx/updateTR`

`POST /tx/updateTR` takes `{ transactionId, destinationAddress, assetType, amount, pii, originatorVASPdid,
beneficiaryVASPdid }`, all required except `pii` (note the different names). Semantics unverified; prefer
`/tx/update`.
