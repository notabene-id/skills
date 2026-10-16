# Send a transfer (originator, v1)

As the originator institution you create a transaction carrying both customers' IVMS 101 data, approve it, wait for the
beneficiary institution to confirm the address and accept, and only then broadcast on-chain and record the hash.
Statuses and the verb table are in [Getting started](getting-started.md); pitfalls in [Caveats](caveats.md).

## Contents

- [Quick start](#quick-start)
- [1. Validate (optional)](#1-validate-optional)
- [2. Create](#2-create)
- [3. Approve](#3-approve)
- [4. Wait for the beneficiary institution](#4-wait-for-the-beneficiary-institution)
- [5. Record the on-chain hash](#5-record-the-on-chain-hash)
- [6. Cancel](#6-cancel)
- [/tx/updateTR](#txupdatetr)

## Quick start

1. `POST /tx/create` with a unique `transactionRef`, the amount in base units, and
   `transactionBlockchainInfo.destination`.
2. If the response is `NEW`, `POST /tx/approve?id=<id>` → `SENT` (skip if an auto-approve rule already did it).
3. Wait for `status: "ACCEPTED"` (beneficiary: `SENT → ACK → ACCEPTED`); stop on `REJECTED` / `DECLINED`.
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
| `pii`, `encrypted` | PII as an IVMS-shaped block or an encrypted string. |
| `protocol`, `targetProtocol`, `notificationEmail` | `targetProtocol: "GTR"` routes to GTR; `notificationEmail` for TRLight. |
| `skipBeneficiaryDataValidation`, `travelRuleBehavior` | Validation and behaviour flags. |
| `onBehalfOfMyself` | The customer sends to their own account at another institution. |
| `blockchainTransactionTimestamp` | When the transfer happened on-chain. |

Validation observed against the real v1 stack:

- **Above threshold**, beneficiary persons are mandatory: without them `400
  ["beneficiaryNameMissing","beneficiaryAccountNumberMissing"]`; a `beneficiary` without `beneficiaryPersons` fails
  the schema.
- **Every** originator person above threshold needs a name, an account number, and a national id, date and place of
  birth, or geographic address; a name alone yields `originatorGeographicAddressMissing` or
  `originatorDateAndPlaceOfBirthMissing`.
- **Below threshold**, it parks at `SAVED` and is never sent.
- Party DIDs must match `^did:[a-zA-Z0-9]*:.*$`.

The response is the transaction at `NEW`, or `SENT` if an auto-approve rule fired. Retrying a timed-out create can
duplicate it; see [Caveats](caveats.md) (idempotency).

## 3. Approve

At `NEW` or `MISSING_BENEFICIARY_DATA`: `POST /tx/approve?id=<id>` (no body) → `SENT`.

## 4. Wait for the beneficiary institution

Watch `notification.transactionUpdated` ([Webhooks](webhooks.md)) or poll `GET /tx/info?id=…`. `SENT`: address not
yet confirmed (a changed `beneficiaryVASPdid` means it was redirected; re-evaluate). `ACK`: confirmed, decision
pending. `ACCEPTED`: go. `REJECTED` / `DECLINED`: do not broadcast.

v1 has no separate "authorized" status: `ACCEPTED` is the beneficiary's positive decision. **Never broadcast before
`ACCEPTED`**: the API doesn't block it, but Notabene's own bridge settles only then.

## 5. Record the on-chain hash

After broadcasting, `POST /tx/update` as the originator (allowed after `ACCEPTED`; the beneficiary may set `txHash`
only in `INCOMPLETE`, `MISSING_BENEFICIARY_DATA`, `SAVED`):

```json
{ "id": "6f1c…-uuid", "txHash": "0x9c1d…" }
```

`/tx/update` also takes `destination`, `origin`, `originatorVASPdid`, `beneficiaryVASPdid`, `beneficiaryVASPname`,
`originatorDid`, `beneficiaryDid`, `originatorRef`, `beneficiaryRef`, `originator`, `beneficiary` (top-level in the
request, unlike the `ivms101` response), `originatorProof`, `beneficiaryProof`, `pii`, `isNonCustodial`, `comment`,
`originatorNameHash`, `beneficiaryNameHash`, `skipBeneficiaryDataValidation`, `notificationEmail`, `protocol`,
`targetProtocol`, `onBehalfOfMyself`, `blockchainTransactionTimestamp`. Party data is accepted only at `INCOMPLETE` /
`MISSING_BENEFICIARY_DATA`. `originatorVASPdid` swaps the originator institution (refused once `ACCEPTED`,
`DECLINED` or `REJECTED`). Omit keys you don't mean to change (`""` clears a field), and re-read to verify (it can
return `200` without effect).

## 6. Cancel

`POST /tx/cancel`, with optional snake_case `vasp_did`:

```json
{ "id": "6f1c…-uuid", "vasp_did": "did:ethr:0xaaaa…", "reason": "customer cancelled withdrawal" }
```

Cancel before the beneficiary acts: it can be refused at `SENT` / `ACK` (seen on deposit-notify transactions), where
only the beneficiary can still end the transaction (`/tx/decline`), and it is illegal after `ACCEPTED`. Re-read the
status first.

## `/tx/updateTR`

`POST /tx/updateTR` takes `{ transactionId, destinationAddress, assetType, amount, pii, originatorVASPdid,
beneficiaryVASPdid }`, all required except `pii` (note the different names). Semantics unverified; prefer
`/tx/update`.
