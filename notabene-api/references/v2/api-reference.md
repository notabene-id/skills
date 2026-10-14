# Notabene API Reference (v2)

OpenAPI 3.1.0, version 1.0.0. Auth, base URL and identifier formats:
[guide-travel-rule-api.md#authentication](guide-travel-rule-api.md#authentication).

## Table of Contents

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

- `{entityDID}` (in almost every path): your institution's DID, e.g. `did:web:vasps.id:jxnl0411:at`. Pattern:
  `^did:([a-z0-9]+):((?:(?:[a-zA-Z0-9._-]|(?:%[0-9a-fA-F]{2}))*:)*(?:[a-zA-Z0-9._-]|(?:%[0-9a-fA-F]{2}))+)((;[a-zA-Z0-9_.:%-]+=[a-zA-Z0-9_.:%-]*)*)(/[^#?]*)?([?][^#]*)?(#.*)?$`
- All errors (400, 401, 403, 404, 409, 500) share `{ "error": "string", "details": "string | string[]" }`.
- Note the path prefixes: most endpoints use `/entities/...`; address-ownership uses `/entity/...` (singular); PII
  endpoints use `/transfers/{id}` where transfer endpoints use `/tx/{id}`.

---

## Address Ownership

### `POST /entity/{entityDID}/address-ownership/discover`

Discover who owns a blockchain address, using relationships, the hashed-address service and blockchain analytics.

Body (one of): `{ "caip10": "eip155:1:0x742E..." }` or `{ "asset": "eip155:1/slip44:60", "address": "0x742E..." }`.

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

`confidence`: `CONFIRMED`, `UNCONFIRMED`, `NOT_FOUND`. `source`: `ADDRESS_BOOK`, `HASHED_ADDRESS`, `INTEGRATION`,
`UNKNOWN`. Errors: 404 (returns `NOT_FOUND` confidence), 501 `NOT_IMPLEMENTED`.

---

## Entities

### `GET /entities/{entityDID}/public-keys`

Returns the entity's encryption key used in webhooks. For legacy v1 VASPs (`did:ethr`) returns the Gateway VASP's keys
and adds `gatewayDID`. Key selection: id ending `#pii` > `#notabene-pii` > type `X25519KeyAgreementKey2019` > first
available. Cached 5 minutes.

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

---

## Network

| Method | Path | Purpose |
|---|---|---|
| GET | `/networks/{entityDID}` | One VASP/entity: regulatory, verification and protocol-support info |
| GET | `/network` | List/search entities, paginated |
| GET | `/networks/gleifSearch` | Look up a Legal Entity Identifier in GLEIF |

**`GET /networks/{entityDID}`** query: `fields` (comma-separated), `reviewedByVaspDID`, `showJurisdictionStatus`,
`includeSubsidiaries`. Response fields include `did`, `name`, `legalName`, `website`, `incorporationCountry`,
`isRegulated`, `regulatoryStatus`, `verificationStatus`, `hasAdmin`, `isNotabeneCustomer`, `subsidiaries[]`,
`badges[]`, `isSandbox`, plus address, Travel Rule protocol support, compliance phase, etc.

**`GET /network`** query: `q`, `emailDomain`, `badge` (`super_vasp`, `verified`, `in_network`, `claimed`),
`jurisdictions`, `page`, `per_page`, `fields`, `listingType` (`exclude_subsidiaries`, `all`, `exclude_gateways`),
`listing` (`sandbox`, `non-sandbox`). Response: `{ "vasps": [...], "pagination": { "page", "per_page", "total" } }`.

**`GET /networks/gleifSearch`** query: `leiNumber` (required, `^([A-Z0-9]){20}$`). Returns legal name, jurisdiction
and registration details.

---

## Relationships

Links an address (`from`, usually a `did:pkh`) to an entity (`to`), optionally with the `custodian` that manages it.

| Method | Path | Purpose | Response |
|---|---|---|---|
| POST | `/entities/{entityDID}/relationships` | Create a relationship, optionally with proof | `201` `{ "relationship": {...} }` |
| GET | `/entities/{entityDID}/relationships` | List; query `from`, `to`, `custodian` | `200` `{ "relationships": [...] }` |
| PATCH | `/entities/{entityDID}/relationships?from=&to=&custodian=` | Confirm (upsert); optional proof body | `201` `{ "success": true }` |
| DELETE | `/entities/{entityDID}/relationships?from=&to=` | Delete (`from`, `to` required) | `202` `{ "success": true }` |

POST body:

| Field | Required | Validation | Description |
|---|---|---|---|
| `to` | yes | 3–255 chars | Owning entity DID |
| `from` | no | 3–255 chars | Address DID |
| `custodian` | no | 3–255 chars, DID pattern | Custodian (wallet service) DID |
| `proof` | no | | Ownership proof, one of the types below |

Proof types: **signature** — `type` in `eip-191`, `eip-712`, `eip-1271`, `bip-137`, `bip-322`, `xpub`, `tip-191`,
`ed25519`, `xrp-ed25519`, `xlm-ed25519`, `cip-8`, `siwe`, `siwx`, with `proof` and `attestation`; **screenshot** —
`url` (URI); **declaration** — `confirmed` (boolean).

Relationship object: `@id`, `from`, `to`, `custodian`, `status` (`UNCONFIRMED`, `CONFIRMED`, `PROVEN`),
`createdTime`, `proofId`; list items also carry `confirmedBy` and `proofs[]`:

```json
{ "status": "VERIFIED", "did": "did:pkh:...", "address": "eip155:1:0x742d35Cc...", "type": "eip191",
  "proof": "0x...", "walletProvider": "MetaMask", "confirmed": true, "createdAt": "2025-01-15T10:00:00.000Z" }
```

Proof `status`: `PENDING`, `CONFIRMED`, `VERIFIED`, `REJECTED`. Proof `type` in list responses: `screenshot`,
`eip191`, `eip712`, `declaration`.

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
| POST | `/entities/{entityDID}/tx/{transferId}/agents` | Add an agent | `202` |
| POST | `/entities/{entityDID}/tx/{transferId}/agents/replace` | Replace an agent | `202` |

### `POST /entities/{entityDID}/tx`

Entry point for Travel Rule compliance. Full examples (hosted, unhosted, unmatched deposit):
[guide-travel-rule-api.md#1-create-a-transfer](guide-travel-rule-api.md#1-create-a-transfer).

| Field | Required | Validation | Description |
|---|---|---|---|
| `ref` | yes | `^[a-zA-Z0-9_-]{1,64}$` | Idempotency key; duplicate → `409` |
| `originator` | yes | `@id` 3–255 chars, valid DID | `{ "@id": "..." }` |
| `beneficiary` | yes | `@id` 3–255 chars, valid DID | `{ "@id": "..." }` |
| `asset` | yes | CAIP-19 or 2–20 char abbreviation | e.g. `BTC` |
| `amount` | yes | `^\d+(\.\d+)?$` (no negatives) | Decimal string |
| `agents` | yes | min 1 | Agent objects (below) |
| `settlementId` | no | max 255, CAIP-220 recommended | On-chain tx hash |
| `transactionValue` | no | `amount` `^\d+(\.\d+)?$`; `currency` 1–3 chars | `{ amount, currency }` or null |
| `blockchainAnalyticsAlias` | no | | Alias for blockchain analytics |
| `memoTag` | no | 1–28 chars, `^[ -~]+$` | Memo/destination tag |

Agent object:

| Field | Required | Description |
|---|---|---|
| `@id` | yes | Agent DID or address |
| `role` | no | `VASP`, `Custodian`, `SettlementAddress`, `SourceAddress`, `Gateway`, `Unknown` (default) |
| `for` | no | DID of the party this agent acts for |
| `policies` | no | `[{ "@type": "REQUIRE_AUTHORIZATION" \| "REQUIRE_PRESENTATION" \| "REQUIRE_RELATIONSHIP_CONFIRMATION" }]` |
| `memoTag` | no | Memo/destination tag for address agents |

### `GET /entities/{entityDID}/tx` and `/tx/search`

| Query | Type | Notes |
|---|---|---|
| `limit` | integer | default 25, 1–100 |
| `offset` | integer | default 0 |
| `agent`, `status`, `asset`, `party`, `refs` | array | Filters |
| `created_before`, `created_after` | string | ISO 8601 |
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
| `sanitize` | `"true"` | Sanitize sensitive data |
| `include_timeline` | `"false"` | Include event timeline |
| `include_integration_results` | `"false"` | Include integration results |
| `include_replaced` | `false` | Include replaced agents |

### `GET /entities/{entityDID}/tx/match`

Query: `settlement_id` (required, tx hash), `settlement_address` (required, destination), `settlement_id_index`
(vout/log index; matched strictly first, dropped if broadened), `memo_tag`, `limit` (25, max 100), `offset`,
`direction`. Response adds `meta: { "match_strategy": "strict" | "broadened" | "none", "broadened_candidates": 0 }`.

### `POST /entities/{entityDID}/tx/{transferId}/authorize`

Query: `force` (authorize even if checks fail), `ignoreFlags` (comma-separated flags to ignore). Body (both optional):

| Field | Validation | Description |
|---|---|---|
| `settlementAddress` | `^[a-z0-9-]+:[a-z0-9-]+:.+$` (CAIP-10) | Settlement address |
| `memoTag` | 1–28 printable ASCII | Memo/destination tag |

Response `202` `{ "message": "Transfer authorization accepted" }`. Wallet services authorize with `{}`
([wallet-service-guide.md](wallet-service-guide.md#common-pitfalls)).

### `POST /entities/{entityDID}/tx/{transferId}/reject`

Body `{ "reason": "...", "comment": "..." }`; `comment` max 500, required when `reason` is `OTHER`. Reason enum:
[guide-travel-rule-api.md#3-reject-a-transfer](guide-travel-rule-api.md#3-reject-a-transfer). Response `200`
`{ "message": "Transfer rejected successfully" }`.

### `POST /entities/{entityDID}/tx/{transferId}/settle`

Body (all optional): `settlementId` (tx hash), `settlementIdIndex` (e.g. `"0"`), `revertSettlementId`. Response `202`
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

---

## TAP Policies

| Method | Path | Response |
|---|---|---|
| GET | `/entities/{entityDID}/transfers/{transferId}/tap-policies` | `{ "policies": [TapPolicy] }` |
| GET | `/entities/{entityDID}/transfers/{transferId}/tap-policies/{policyId}` | `{ "policy": TapPolicy }` |

```json
{
  "@id": "cb16e827-a11b-4b59-8fc0-9c36459b9447",
  "@type": "REQUIRE_PRESENTATION",
  "status": "PENDING",
  "from": "did:web:counterparty.com",
  "for": "did:web:your-vasp.com",
  "about": "originator",
  "presentationDefinition": "https://...",
  "encryptionKey": { "id": "did:web:counterparty.com#pii", "type": "EcdsaSecp256r1VerificationKey2019",
                     "controller": "did:web:counterparty.com", "publicKeyHex": "04a1b2..." },
  "createdAt": "2025-12-01T12:24:52.537Z"
}
```

`@type`: `REQUIRE_AUTHORIZATION`, `REQUIRE_PRESENTATION`, `REQUIRE_RELATIONSHIP_CONFIRMATION`. `status`:
`COMPLETED`, `INCOMPLETE`, `PENDING`.

---

## Transfer Checks

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

### `POST /entities/{entityDID}/transfers/{transferId}/append`

Append IVMS101 PII (Notabene encrypts it). Body: `originator` / `beneficiary` (`{ "@id" }`) and `ivms101` with
`originator.originatorPerson[]` + `accountNumber`, and/or `beneficiary.beneficiaryPerson[]` + `accountNumber`
([IVMS101](#ivms101)). Example: [guide-travel-rule-api.md#2-provide-pii-when-requested](guide-travel-rule-api.md#2-provide-pii-when-requested).
Response `202` `{ "message": "Transfer appended successfully with encrypted PII" }`.

### `POST /entities/{entityDID}/transfers/{transferId}/policies/{policyId}/presentation`

Present PII for one policy. Query `skipValidation` (boolean, default false). Body `{ "ivms101": <IVMS101> }`. Response
`202` `{ "message": "Presentation accepted and processing" }`.

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
| `agents` | array | yes | Transfer agents |
| `amount` | string? | no | Amount |
| `originator` | object? | no | `{ "@id", "originatorPerson": [...] }` |
| `beneficiary` | object? | no | `{ "@id", "beneficiaryPerson": [...] }` |
| `ref` | string? | no | Client reference |
| `direction` | string? | no | `INCOMING`, `OUTGOING` |
| `transactionType` | string? | no | `TRANSFER`, `PAYOUT`, `PAYIN` |
| `settlementAddress` | string? | no | Settlement address |
| `settlementId` | string? | no | On-chain tx hash |
| `settlementIdIndex` | string? | no | Vout/log index |
| `isTravelRule` | boolean? | no | Whether the Travel Rule applies |
| `memoTag` | string? | no | Memo/destination tag |
| `createdTime`, `authorizedTime`, `rejectedTime`, `settledTime`, `revertedTime`, `expiresTime` | string? | no | Timestamps |
| `amounts` | array? | no | Amounts with currency conversions |
| `flags` | array? | no | Transfer flags |
| `log` | array? | no | Activity log |
| `policyEvaluations` | array? | no | Policy evaluation results |

Agent status values: `PROCESSING`, `AUTHORIZED`, `REJECTED`, `SETTLED`, `FAILED`, `REPLACED`. Agent roles: `VASP`,
`Custodian`, `SettlementAddress`, `SourceAddress`, `Gateway`, `Unknown`.

### IVMS101

IVMS 101.2023 structure for originator and beneficiary PII.

```
ivms101:
  originator:
    originatorPerson[]:
      naturalPerson:
        name (REQUIRED):
          nameIdentifier[] (REQUIRED):
            primaryIdentifier (REQUIRED): string
            secondaryIdentifier: string
            naturalPersonNameIdentifierType (REQUIRED): string (e.g., "LEGL")
        geographicAddress[]:
          addressType (REQUIRED): string
          addressLine[]: string  (use this OR streetName+buildingNumber)
          streetName: string
          buildingNumber: string
          postCode: string
          townName (REQUIRED): string
          country (REQUIRED): string (ISO-3166 Alpha-2)
        nationalIdentification:
          nationalIdentifier (REQUIRED): string
          nationalIdentifierType (REQUIRED): string
          registrationAuthority: string
          countryOfIssue: string (ISO-3166 Alpha-2)
        customerIdentification: string
        dateAndPlaceOfBirth:
          dateOfBirth (REQUIRED): string (YYYY-MM-DD)
          placeOfBirth (REQUIRED): string
      legalPerson:
        name (REQUIRED):
          nameIdentifier[] (REQUIRED):
            legalPersonName (REQUIRED): string
            legalPersonNameIdentifierType (REQUIRED): string
        geographicAddress[]: (same structure)
        nationalIdentification:
          nationalIdentifier (REQUIRED): string
          nationalIdentifierType (REQUIRED): string
          registrationAuthority (REQUIRED for beneficiary): string
        customerIdentification: string
      accountNumber[]: string
  beneficiary:
    beneficiaryPerson[]: (same structure as originatorPerson)
```
