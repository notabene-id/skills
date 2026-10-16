# Notabene API Reference (v2)

OpenAPI 3.1.0, version 1.0.0. Auth, base URL and identifier formats:
[guide-travel-rule-api.md#authentication](guide-travel-rule-api.md#authentication).

## Contents

- [Conventions](#conventions)
- [Address Ownership](#address-ownership)
- [Entities](#entities)
- [Network](#network)
- [Relationships](#relationships)
- [Transfers](#transfers)
- [TAP Policies](#tap-policies)
- [Transfer Checks](#transfer-checks)
- [PII](#pii)
- [Schemas](#schemas)

---

## Conventions

- `{entityDID}` (in almost every path): your institution's DID, e.g. `did:web:vasps.id:jxnl0411:at`. Must be a valid
  DID (`did:<method>:<id>`, 3–255 chars); URL-encode it in the path.
- All errors (400, 401, 403, 404, 409, 500, 503) share `{ "error": "string", "details": "string | string[]" }`.
- Use the paths below. Deprecated aliases are still served: `/entities/{entityDID}/transfers/...` for most `/tx/...`
  transfer routes, `/entities/{entityDID}/relationship` for `/relationships`, and a leading `/entity/...` (rewritten
  to `/entities/...`). Don't build new code on them.

---

## Address Ownership

### `POST /entities/{entityDID}/address-ownership/discover`

Discover who owns a blockchain address, using relationships (address book) and the hashed-address service.

Body (both fields required): `{ "asset": "eip155:1/slip44:60", "address": "0x742E..." }`. `asset`: CAIP-19, crypto
symbol (e.g. `ETH`) or DTI. `address`: plain address or CAIP-10, 1–256 chars.

Response `200`:

```json
{
  "addressOwnership": {
    "address": "eip155:1:0x742d35Cc...",
    "asset": "ETH",
    "agent": { "did": "did:web:entity1.example.com", "name": "Example Entity 1", "jurisdiction": "US" },
    "custodian": { "did": "did:web:fireblocks.com", "name": "Fireblocks", "jurisdiction": "US" },
    "relationship": { "status": "Confirmed", "confirmedAt": "2025-01-15T10:00:00.000Z",
                      "confirmedBy": "did:web:example.com" },
    "confidence": "CONFIRMED",
    "source": "ADDRESS_BOOK",
    "discoveredAt": "2025-01-15T10:30:00.000Z",
    "responseTimeMs": 125
  },
  "meta": { "requestId": "550e8400-...", "timestamp": "2025-01-15T10:30:00.000Z" }
}
```

`confidence`: `CONFIRMED`, `UNCONFIRMED`, `NOT_FOUND`. `source`: `ADDRESS_BOOK`, `ADDRESS_HASH`, `UNKNOWN`.
`relationship` may also carry `proofs[]`, and `memoTag` appears for memo-tag chains. Errors: 400, 401, 403, 404 (when
confidence is `NOT_FOUND`; the error `data` still carries `addressOwnership` and `meta`), 500.

---

## Entities

### `GET /entities/{entityDID}/public-keys`

Returns the entity's encryption key used in webhooks. For legacy v1 VASPs (`did:ethr`) returns the Gateway VASP's keys
and adds `gatewayDID`. Key selection: id ending `#pii` > `#notabene-pii` > `#nb-pii` > P-256 `JsonWebKey2020` keys
referenced from `keyAgreement` > type `X25519KeyAgreementKey2019` > first available. DID resolution is cached for 5
minutes. Any entity's keys can be read.

```json
{
  "entityDID": "did:web:entity2.example.com",
  "encryptionKey": {
    "id": "did:web:entity2.example.com#pii",
    "type": "EcdsaSecp256r1VerificationKey2019",
    "controller": "did:web:entity2.example.com",
    "publicKeyHex": "04a1b2c3d4e5f6...",
    "publicKeyBase58": "2Abc3Def4Ghi5Jkl..."
  },
  "gatewayDID": "did:web:gateway.notabene.studio"
}
```

Errors: 404, 500, 503 (DID resolution failed).

### `POST /entities/{entityDID}/delegateToken`

Issues a short-lived (1 hour) token for one end customer, for the JavaScript SDK's front-end components
([guide-javascript-sdk.md](guide-javascript-sdk.md#authentication-customer-delegate-token)). Call it server-side.

Body `{ "entityDID": "<same as path>", "delegateId": "<your customer id, 1–255 chars>" }`. Response `200`
`{ "access_token": "...", "token_type": "Bearer", "delegateDid": "did:..." }`; the same `delegateId` always maps to
the same `delegateDid`.

---

## Network

| Method | Path | Purpose |
|---|---|---|
| GET | `/network/{entityDID}` | One VASP/entity: regulatory, verification and protocol-support info |
| GET | `/network` | List/search entities, paginated |
| GET | `/network/gleifSearch` | Look up a Legal Entity Identifier in GLEIF |

**`GET /network/{entityDID}`** query: `fields` (comma-separated), `reviewedByVaspDID` (DID), `showJurisdictionStatus`,
`includeSubsidiaries`, `networkStatus`, `regulatoryStatus` (each a string or string array). Response fields include
`did`, `name`, `legalName`, `website`, `incorporationCountry`, `isRegulated`, `verificationStatus`, `hasAdmin`,
`isNotabeneCustomer`, `subsidiaries[]`, `badges[]`, `isSandbox`, plus address, Travel Rule protocol support,
compliance phase, etc.

**`GET /network`** query: `q`, `emailDomain`, `badge` (`super_vasp`, `verified`, `in_network`, `claimed`),
`jurisdictions`, `page`, `per_page`, `order` (sort field, e.g. `name`), `fields`, `listingType`
(`exclude_subsidiaries`, `all`, `exclude_gateways`), `listing` (`sandbox`, `non-sandbox`), `regulatoryStatus`
(`Regulated`, `Not Regulated`, `Exempt`, `Pending`), `status` (`unreachable`, `receive`, `receive_respond`,
`receive_respond_send`; pipe-separated for OR). Response:
`{ "vasps": [...], "pagination": { "page": 0, "per_page": 100, "order": "name", "total": 100 } }`.

**`GET /network/gleifSearch`** query: `leiNumber` (required, `^([A-Z0-9]){20}$`). Response `{ "results": [...] }`;
each result has `provider` (`gleif`), `externalId`, `entity`, `registration`, `updatedAt`, `resolvedAt`.

---

## Relationships

Links an address (`from`, usually a `did:pkh`) to an entity (`to`), optionally with the `custodian` that manages it.

| Method | Path | Purpose | Response |
|---|---|---|---|
| POST | `/entities/{entityDID}/relationships` | Create a relationship, optionally with proof | `201` `{ "relationship": {...} }` |
| GET | `/entities/{entityDID}/relationships` | List; query `from`, `to`, `custodian`, `asset` | `200` `{ "relationships": [...] }` |
| PATCH | `/entities/{entityDID}/relationships?from=&to=&custodian=&asset=` | Confirm (upsert); `from` and `to` required (query, or body for backwards compatibility); optional body `{ "proof": {...} }` | `201` `{ "success": true }` |
| DELETE | `/entities/{entityDID}/relationships?from=&to=&asset=` | Delete (`from`, `to` required) | `202` `{ "success": true }` |

A relationship is confirmed on creation when your `entityDID` is its `from`, `to` or `custodian`. `asset` (CAIP-19)
normalizes a plain `from` address to `did:pkh`.

POST body:

| Field | Required | Validation | Description |
|---|---|---|---|
| `to` | yes | 3–255 chars | Owning entity DID |
| `from` | no | 3–255 chars; a `did:pkh:` value must hold a valid CAIP-10 address | Address DID; defaults to `entityDID` |
| `custodian` | no | valid DID, 3–255 chars | Custodian (wallet service) DID |
| `proof` | no | | Ownership proof, one of the types below |
| `asset` | no | | Asset used to normalize a plain `from` address to `did:pkh` |

Every proof requires `status` (`pending`, `rejected`, `flagged`, `verified`), `did` (valid DID) and `address`
(CAIP-10; a `did:pkh:` prefix is stripped); `xpub` and `chainSpecificData` are optional. Proof types:
**signature** — `type` in `eip-191`, `eip-712`, `eip-1271`, `bip-137`, `bip-322`, `tip-191`, `ed25519`,
`xrp-ed25519`, `xlm-ed25519`, `cip-8`, `siwe`, `siwx`, with `proof` and `attestation`, optional `wallet_provider`;
**screenshot** — `url` (URI); **self-declaration** — `type: "self-declaration"`, `confirmed` (boolean), optional
`attestation`; **micro-transfer** — `type: "microtransfer"`, optional `proof`, `chain`, `destination`,
`amountSubunits`.

Relationship object: `@id`, `from`, `to`, `custodian`, `status` (`UNCONFIRMED`, `CONFIRMED`, `PROVEN`), `memoTag`,
`createdTime`, `updatedTime`, `proofId`; list items also carry `confirmedBy` and `proofs[]` with the same `status` and
`type` values as above:

```json
{ "status": "verified", "did": "did:pkh:...", "address": "eip155:1:0x742d35Cc...", "type": "eip-191",
  "proof": "0x...", "attestation": "...", "wallet_provider": "MetaMask" }
```

---

## Transfers

| Method | Path | Purpose | Success |
|---|---|---|---|
| POST | `/entities/{entityDID}/tx` | Create a transfer | `201` `{ "transfer": Transfer }` |
| GET | `/entities/{entityDID}/tx` | List, filter, paginate | `200` `{ "data": [], "pagination": { total, limit, offset } }` |
| GET | `/entities/{entityDID}/tx/{transferId}` | Get one (optionally decrypted) | `200` `{ "transfer": Transfer }` |
| GET | `/entities/{entityDID}/tx/search` | Search; items carry `matchedColumns` | `200` same shape as list |
| GET | `/entities/{entityDID}/tx/match` | Match by on-chain settlement data | `200` list + `meta` |
| POST | `/entities/{entityDID}/tx/{transferId}/authorize` | Authorize | `202` |
| POST | `/entities/{entityDID}/tx/{transferId}/reject` | Reject | `200` |
| POST | `/entities/{entityDID}/tx/{transferId}/settle` | Report on-chain settlement | `202` |
| POST | `/entities/{entityDID}/tx/{transferId}/append` | Append IVMS101 PII ([PII](#pii)) | `202` |
| POST | `/entities/{entityDID}/tx/{transferId}/agents` | Add an agent | `202` |
| POST | `/entities/{entityDID}/tx/{transferId}/agents/replace` | Replace an agent | `202` |
| POST | `/entities/{entityDID}/tx/{transferId}/agents/remove` | Remove an agent | `202` |

### `POST /entities/{entityDID}/tx`

Entry point for Travel Rule compliance. Examples: hosted and unhosted wallet in
[guide-travel-rule-api.md#1-create-a-transfer](guide-travel-rule-api.md#1-create-a-transfer); unmatched deposit in
[guide-travel-rule-api.md#4-reconcile-received-transfers](guide-travel-rule-api.md#4-reconcile-received-transfers).

| Field | Required | Validation | Description |
|---|---|---|---|
| `ref` | yes | `^[a-zA-Z0-9_-]{1,255}$` | Idempotency key; duplicate → `409` (`Transfer with ref … already exists`) |
| `originator` | yes | `@id` 3–255 chars, an IRI (`mailto:…`, `did:…`); a `did:` value must be a valid DID | `{ "@id": "..." }` |
| `beneficiary` | yes | same as `originator` | `{ "@id": "..." }` |
| `asset` | yes | CAIP-19, uppercase symbol of 2–20 chars (`[A-Z0-9-]`, e.g. `BTC`), or 10-char DTI | Asset identifier |
| `amount` | yes | `^\d+(\.\d+)?$` (no negatives) | Decimal string |
| `agents` | yes | min 1 | Agent objects (below) |
| `settlementId` | no | CAIP-220 (`eip155:1:tx/0x…`); raw hashes are normalized when the chain is known from the asset | On-chain tx hash |
| `transactionValue` | no | `amount` `^\d+(\.\d+)?$`; `currency` 1–3 chars | `{ amount, currency }` or null |
| `blockchainAnalyticsAlias` | no | | Alias for blockchain analytics |
| `memoTag` | no | 1–28 chars, `^[ -~]+$` | Memo/destination tag |

Agent object:

| Field | Required | Description |
|---|---|---|
| `@id` | yes | 2–255 chars. A valid DID, except for `SourceAddress` / `SettlementAddress`, which also accept a plain or CAIP-10 address (normalized to `did:pkh`) |
| `role` | no | `VASP`, `Custodian`, `SettlementAddress`, `SourceAddress`, `Gateway`, `Unknown` (default) |
| `for` | no | 2–255 chars. Identifier of the party or agent this agent acts for |
| `policies` | no | min 1. `[{ "@type": "REQUIRE_AUTHORIZATION" \| "REQUIRE_PRESENTATION" \| "REQUIRE_RELATIONSHIP_CONFIRMATION" }]` |
| `memoTag` | no | 1–28 printable ASCII chars. Only on `SourceAddress` / `SettlementAddress` agents |
| `selectedAsset` | no | Asset selected by this agent (CAIP-19 or ISO currency) |
| `selectedSettlementAddress` | no | Settlement address selected by this agent |

The create response carries only creation-time fields plus `requestForInformation` (`{ isRFI, direction,
actionRequired }`); lifecycle fields such as `isTravelRule`, `settlementAddress` and the `*Time` timestamps appear on
GET.

### `GET /entities/{entityDID}/tx` and `/tx/search`

| Query | Type | Notes |
|---|---|---|
| `limit` | integer | default 25, 1–100 |
| `offset` | integer | default 0 |
| `agent`, `status`, `asset`, `party` | array | Filters, array syntax (`agent[]=…`); `status` takes the [Transfer](#transfer) status values |
| `refs` | string or array | Filter by reference |
| `created_before`, `created_after` | string | Date starting `YYYY-MM-DD` (time optional) |
| `direction` | string | `INCOMING`, `OUTGOING` |
| `search_term` | string | Full-text |
| `fields` | string | Comma-separated |
| `sort_by` / `sort_order` | string | `asc`, `desc` |
| `action_required` | boolean | Transfers waiting on you |
| `request_for_information` | boolean | Pending info requests |

`/tx/search` takes the same parameters and returns `SearchTransfer` items with `matchedColumns`
(e.g. `["ref", "settlementId"]`).

### `GET /entities/{entityDID}/tx/{transferId}`

| Query | Default | Description |
|---|---|---|
| `decrypt` | `"false"` | `"true"` decrypts PII |
| `sanitize` | `"false"` | Sanitize sensitive data |
| `include_timeline` | `"false"` | Include event timeline |
| `include_integration_results` | `"false"` | Include integration results |
| `include_replaced` | `"false"` | Include replaced agents |
| `include_removed` | `"false"` | Include removed agents |
| `include_comments` | `"false"` | Include comments |

### `GET /entities/{entityDID}/tx/match`

Query: `settlement_id` (required; CAIP-220 `eip155:1:tx/0x…`, or a raw hash normalized when the chain is known),
`settlement_address` (required, destination: plain, CAIP-10 or `did:pkh`), `source_address` (sender, CAIP-10; an
embedded memo tag is used for matching), `asset` (CAIP-19, needed only when `settlement_address`, `source_address` and
`settlement_id` are all plain), `settlement_id_index` (vout/log index; matched strictly first, dropped if broadened),
`memo_tag`, `limit` (25, max 250), `offset`, `direction`, `fields`, `sort_by`, `sort_order`. Response adds
`meta: { "match_strategy": "strict" | "broadened" | "none", "broadened_candidates": 0 }`.

### `POST /entities/{entityDID}/tx/{transferId}/authorize`

Query: `force` (`"true"` bypasses certain checks), `ignoreFlags` (`"true"` ignores flag policies). Body required, `{}`
is valid; both fields optional and any other field is ignored:

| Field | Validation | Description |
|---|---|---|
| `settlementAddress` | CAIP-10 (`eip155:1:0x…`) or RFC 8905 `payto://` URI | Settlement address (beneficiary side) |
| `memoTag` | string | Memo/destination tag |

Response `202` `{ "message": "Transfer authorization accepted" }`. Wallet services authorize with `{}`
([wallet-service-guide.md](wallet-service-guide.md#common-pitfalls)).

### `POST /entities/{entityDID}/tx/{transferId}/reject`

Body `{ "reason": "...", "comment": "..." }`; `comment` max 500, required when `reason` is `OTHER`. Reason enum:
[guide-travel-rule-api.md#3-reject-a-transfer](guide-travel-rule-api.md#3-reject-a-transfer). Response `200`
`{ "message": "Transfer rejected successfully" }`.

### `POST /entities/{entityDID}/tx/{transferId}/settle`

Body (all optional): `settlementId` (tx hash, max 255, CAIP-220 recommended), `settlementIdIndex` (vout/log index,
max 64, e.g. `"0"`), `revertSettlementId` (max 255, CAIP-220 recommended). Response `202`
`{ "message": "Transfer settlement accepted" }`.

### `POST /entities/{entityDID}/tx/{transferId}/agents`

Body `{ "agent": { "@id", "role", "for", "policies", "memoTag" } }` (agent object above). Response `202`.

### `POST /entities/{entityDID}/tx/{transferId}/agents/replace`

Caller must be the transfer initiator, a gateway, or have a relationship with the agent being replaced.

```json
{ "original": "did:web:old-agent.com",
  "replacement": { "@id": "did:web:new-agent.com", "role": "VASP", "for": "mailto:user@example.com" } }
```

Response `202`.

### `POST /entities/{entityDID}/tx/{transferId}/agents/remove`

Removes (soft-deletes) an agent. Same caller rule as replace. Body
`{ "original": "did:web:agent.com", "reason": "..." }` (`reason` optional, max 500 chars). Response `202`
`{ "message": "Agent removal accepted" }`; the transfer's parties
receive `notification.transferAgentRemoved` ([webhooks.md](webhooks.md#transfer-management-events)).

---

## TAP Policies

| Method | Path | Response |
|---|---|---|
| GET | `/entities/{entityDID}/tx/{transferId}/tap-policies` | `{ "policies": [TapPolicy] }`, the policies on all of the transfer's agents |
| GET | `/entities/{entityDID}/tx/{transferId}/tap-policies/{policyId}` | `{ "agent": {...} }`: one policy wrapped in the agent that owns it |

List item:

```json
{
  "@id": "cb16e827-a11b-4b59-8fc0-9c36459b9447",
  "@type": "REQUIRE_PRESENTATION",
  "status": "PENDING",
  "from": "did:web:counterparty.com",
  "for": "did:web:your-vasp.com",
  "fromRole": "...",
  "fromAgent": "...",
  "presentationDefinition": "https://...",
  "createdAt": "2025-12-01T12:24:52.537Z",
  "updatedAt": "2025-12-01T12:24:52.537Z"
}
```

Single policy:

```json
{
  "agent": {
    "@id": "did:web:counterparty.com", "@type": "TransferAgent", "name": "Counterparty VASP", "for": "...",
    "policy": {
      "@id": "...", "@type": "REQUIRE_PRESENTATION", "status": "PENDING", "from": "...", "for": "...",
      "presentationDefinition": "https://...", "purpose": "...", "about": "...", "aboutParty": "...",
      "aboutAgent": "...", "nonce": "...",
      "encryptionKey": { "id": "did:web:...#pii", "type": "...", "controller": "...", "publicKeyHex": "..." }
    }
  }
}
```

`encryptionKey` is the key of the policy's `for` entity, included only for `REQUIRE_PRESENTATION` policies that are
for another entity. `@type`: `REQUIRE_AUTHORIZATION`, `REQUIRE_PRESENTATION`, `REQUIRE_RELATIONSHIP_CONFIRMATION`.
`status`: `COMPLETED`, `INCOMPLETE`, `PENDING`, `REPLACED`.

---

## Transfer Checks

These self-checks exist on a transfer only when the entity has configured them (for beneficiaries with a
`REQUIRE_PRESENTATION` policy); `tap.requirePresentationSatisfied` carries a callback URL for each configured check.

| Method | Path | Marks complete |
|---|---|---|
| POST | `/entities/{entityDID}/tx/{transferId}/checks/beneficiary-name-matching` | Beneficiary name matches your KYC |
| POST | `/entities/{entityDID}/tx/{transferId}/checks/internal-checks` | Operational checks on the PII done |

No body. Response `200`:

```json
{ "id": "uuid", "transferId": "uuid", "entityDid": "did:web:...",
  "type": "REQUIRE_BENEFICIARY_NAME_MATCHING", "status": "COMPLETE",
  "policyData": { "confirmedBy": "did:web:...", "confirmedAt": "2025-01-15T10:00:00.000Z" },
  "createdAt": "...", "updatedAt": "..." }
```

`type` is `REQUIRE_INTERNAL_CHECKS` for internal checks.

---

## PII

### `POST /entities/{entityDID}/tx/{transferId}/append`

Append IVMS101 PII (Notabene-managed encryption). The PII is validated against the transfer's presentation
definition, if set, and encrypted before storage. Body:

| Field | Required | Description |
|---|---|---|
| `ivms101` | yes | `originator.originatorPerson[]` + `accountNumber[]`, and/or `beneficiary.beneficiaryPerson[]` + `accountNumber[]` ([IVMS101](#ivms101)) |
| `originator` / `beneficiary` | no | `{ "@id" }` customer references; giving both enables PII reuse |
| `agents` | no | Agent objects ([above](#post-entitiesentitydidtx)) to add alongside the PII (max 25) |

Examples (originator and beneficiary side):
[guide-travel-rule-api.md#2-provide-pii-when-requested](guide-travel-rule-api.md#2-provide-pii-when-requested).
Response `202` `{ "message": "Transfer appended successfully with encrypted PII" }`.

### `POST /entities/{entityDID}/tx/{transferId}/policies/{policyId}/presentation`

Present PII for one policy (`POST .../tx/{transferId}/presentation` presents for every `REQUIRE_PRESENTATION` policy).
Use these routes when the webhook's `callbackUrl` points at them (entity-managed encryption); a policy that is `for`
your own entity must be fulfilled with `append` instead. Query `skipValidation` (`"true"` skips validation; default
`false`). Body `{ "ivms101": <IVMS101> }`. Response `202` `{ "message": "Presentation accepted and processing" }`
(a retry of a completed policy also answers `202`).

---

## Schemas

### Transfer

| Field | Type | Required | Description |
|---|---|---|---|
| `@id` | string | yes | Transfer identifier |
| `id` | string | no | Alternate ID |
| `status` | string | yes | [Transfer statuses](guide-travel-rule-api.md#transfer-statuses) |
| `asset` | string? | no | CAIP-19 |
| `initiator` | string | yes | DID of the initiating entity |
| `agents` | array | yes | Transfer agents. Each has `@id` (the TransferAgent's own id), `agent: { "@id": <agent DID or address> }`, `role`, `for`, `status`, `policies`, `relationship`, and optional `name`, `memoTag`, `replacement` |
| `amount` | string? | no | Amount |
| `originator` | object? | no | `{ "@id", "originatorPerson": [...] }` |
| `beneficiary` | object? | no | `{ "@id", "beneficiaryPerson": [...] }` |
| `ref` | string? | no | Client reference |
| `direction` | string? | no | `INCOMING`, `OUTGOING` |
| `transactionType` | string? | no | `TRANSFER`, `PAYOUT`, `PAYIN` |
| `settlementAddress` | string? | no | Settlement address (CAIP-10) |
| `settlementId` | string? | no | On-chain tx hash (CAIP-220) |
| `settlementIdIndex` | string? | no | Vout/log index |
| `isTravelRule` | boolean? | no | Whether the Travel Rule applies |
| `internalAuthorization` | string? | no | Your own decision: `PENDING`, `AUTHORIZED`, `REJECTED` |
| `authorizationRequested` | boolean? | no | Whether authorization has been requested |
| `memoTag` | string? | no | Memo/destination tag |
| `createdTime`, `authorizedTime`, `rejectedTime`, `settledTime`, `revertedTime`, `expiresTime` | string? | no | Timestamps |
| `amounts` | array? | no | Amounts with currency conversions |
| `flags` | array? | no | Transfer flags |
| `log` | array? | no | Activity log |
| `policyEvaluations` | array? | no | Policy evaluation results |
| `createdAt`, `updatedAt` | string? | no | Record timestamps |
| `originator.encryptedIvms101`, `beneficiary.encryptedIvms101` | string? | no | Compact JWE of the party's IVMS101, present only when it was sealed to a key under your DID that the node does not hold |

Agent status values: `PROCESSING`, `AUTHORIZED`, `REJECTED`, `SETTLED`, `FAILED`, `REPLACED`, `UNREACHABLE`.
Agent roles: `VASP`, `Custodian`, `SettlementAddress`, `SourceAddress`, `Gateway`, `Unknown`.

### IVMS101

IVMS 101 data for originator and beneficiary PII (v2 field names: `originatorPerson`, `beneficiaryPerson`). The API
schema makes every field optional but rejects unknown keys, caps every array at 25 items, and allows a person to be a
`naturalPerson` or a `legalPerson`, not both. Which fields must be present is decided by the transfer's presentation
definition (the counterparty's jurisdiction); `(IVMS)` marks fields IVMS101 itself requires when their parent is
present. Complete example:
[guide-travel-rule-api.md#2-provide-pii-when-requested](guide-travel-rule-api.md#2-provide-pii-when-requested).

```
ivms101:
  originator:
    originatorPerson[]:
      naturalPerson:
        name:
          nameIdentifier[]:            (also localNameIdentifier[], phoneticNameIdentifier[])
            primaryIdentifier (IVMS): string, max 100
            secondaryIdentifier: string, max 100
            naturalPersonNameIdentifierType (IVMS): string (e.g. "LEGL")
        geographicAddress[]:
          addressType (IVMS): string (e.g. "HOME")
          addressLine[]: string, max 70, up to 7  (use this OR streetName+buildingNumber)
          streetName, buildingNumber, buildingName, postCode, townName, districtName,
          countrySubDivision, department, subDepartment, floor, postBox, room, townLocationName
          country (IVMS): string (ISO-3166 alpha-2; alpha-3 accepted)
        nationalIdentification:
          nationalIdentifier (IVMS): string, max 35
          nationalIdentifierType (IVMS): string
          countryOfIssue: string
          registrationAuthority: string, max 20
        customerIdentification: string, max 50
        dateAndPlaceOfBirth:
          dateOfBirth: string (YYYY-MM-DD)
          placeOfBirth: string, max 70
        countryOfResidence: string
      legalPerson:
        name:
          nameIdentifier[]:
            legalPersonName (IVMS): string, max 255
            legalPersonNameIdentifierType (IVMS): string (e.g. "LEGL")
        geographicAddress[]: (same structure)
        nationalIdentification: (same structure)
        customerIdentification: string, max 50
        countryOfRegistration: string
    accountNumber[]: string, max 255
  beneficiary:
    beneficiaryPerson[]: (same structure as originatorPerson)
    accountNumber[]: string, max 255
```
