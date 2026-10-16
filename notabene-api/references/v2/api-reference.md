# Notabene API Reference

**OpenAPI**: 3.1.0 | **Version**: 1.0.0 | **Base URL**: `https://api.eu1.notabene.id`

Paths are shown in their canonical form. `/entity/...` is an alias of `/entities/...`, and the older
`/entities/{entityDID}/transfers/...` and `/entities/{entityDID}/relationship` paths are deprecated aliases of
`/tx/...` and `/relationships`.

---

## Authentication

All endpoints require a Bearer JWT token obtained via OAuth2 client credentials:

```
POST https://auth.notabene.id/oauth/token
{
  "client_id": "<your_client_id>",
  "client_secret": "<your_client_secret>",
  "grant_type": "client_credentials",
  "audience": "https://api.eu1.notabene.id"
}
```

All requests must include: `Authorization: Bearer <token>`

---

## Table of Contents

- [Address Ownership](#address-ownership)
- [Entities](#entities)
- [Network](#network)
- [Relationships](#relationships)
- [Transfers](#transfers)
- [Transfer Checks](#transfer-checks)
- [PII (Personal Identifiable Information)](#pii)
- [Schemas](#schemas)

---

## Common Path Parameter

Almost every endpoint includes `{entityDID}` in the path:

| Name | Type | Description | Example |
|---|---|---|---|
| `entityDID` | `string (did)` | Your organization's DID | `did:web:vasps.id:jxnl0411:at` |

Must be a valid DID (`did:<method>:<id>`, 3–255 chars; URL-encode it in the path).

## Common Error Response

All error responses (400, 401, 403, 404, 409, 500, 503) share this schema:

```json
{
  "error": "string (required)",
  "details": "string | string[]"
}
```

---

## Address Ownership

### `POST /entities/{entityDID}/address-ownership/discover`

Discover who owns a blockchain address, using relationships (address book) and the hashed address service.

**Request Body** (required, both fields required):

```json
{
  "asset": "eip155:1/slip44:60",
  "address": "0x742E4C6C1c6683A7E1Ba0BA3F8BC43AeDD5b5F4A"
}
```

| Field | Validation |
|---|---|
| `asset` | CAIP-19, crypto symbol (e.g. `ETH`) or DTI |
| `address` | Plain address or CAIP-10, 1–256 chars |

**Response** `200`:

```json
{
  "addressOwnership": {
    "address": "eip155:1:0x742d35Cc...",
    "asset": "ETH",
    "agent": {
      "did": "did:web:entity1.example.com",
      "name": "Example Entity 1",
      "jurisdiction": "US"
    },
    "custodian": {
      "did": "did:web:fireblocks.com",
      "name": "Fireblocks",
      "jurisdiction": "US"
    },
    "relationship": {
      "status": "Confirmed",
      "confirmedAt": "2025-01-15T10:00:00.000Z",
      "confirmedBy": "did:web:example.com"
    },
    "confidence": "CONFIRMED",
    "source": "ADDRESS_BOOK",
    "discoveredAt": "2025-01-15T10:30:00.000Z",
    "responseTimeMs": 125
  },
  "meta": {
    "requestId": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2025-01-15T10:30:00.000Z"
  }
}
```

| Field | Type | Description |
|---|---|---|
| `confidence` | string | `CONFIRMED`, `UNCONFIRMED`, or `NOT_FOUND` |
| `source` | string | `ADDRESS_BOOK`, `ADDRESS_HASH`, or `UNKNOWN` |

`relationship` may also carry `proofs[]`, and `memoTag` appears for memo-tag chains.

**Errors**: 400, 401, 403, 404 (when confidence is `NOT_FOUND`; the error `data` still carries `addressOwnership` and `meta`), 500

---

## Entities

### `GET /entities/{entityDID}/public-keys`

Returns the encryption key used in webhooks for the entity. For V1 VASPs (did:ethr DIDs), returns Gateway VASP's keys instead. Key selection priority: keys ending with `#pii` > `#notabene-pii` > `#nb-pii` > P-256 `JsonWebKey2020` keys referenced from `keyAgreement` > type `X25519KeyAgreementKey2019` > first available. DID resolution is cached for 5 minutes. Any entity's keys can be read.

**Response** `200`:

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

`gatewayDID` is only present for V1 VASPs.

**Errors**: 404, 500, 503 (DID resolution failed)

---

### `POST /entities/{entityDID}/delegateToken`

Issue a short-lived (1 hour) token for one end customer, for the JavaScript SDK's front-end components. Call it
server-side.

**Request Body**: `{ "entityDID": "<same as path>", "delegateId": "<your customer id, 1–255 chars>" }`

**Response** `200`: `{ "access_token": "...", "token_type": "Bearer", "delegateDid": "did:..." }`. The same
`delegateId` always maps to the same `delegateDid`.

---

## Network

### `GET /network/{entityDID}`

Retrieve VASP/entity information including regulatory status, verification status, and protocol support.

**Query Parameters:**

| Name | Type | Description |
|---|---|---|
| `fields` | string | Comma-separated list of fields to include |
| `reviewedByVaspDID` | string (DID) | Filter by reviewing VASP |
| `showJurisdictionStatus` | string | Include jurisdiction status |
| `includeSubsidiaries` | string | Include subsidiary entities |
| `networkStatus` | string or string[] | Filter by network status |
| `regulatoryStatus` | string or string[] | Filter by regulatory status |

**Response** `200`: Returns a VASP entity object with fields including:

| Field | Type | Description |
|---|---|---|
| `did` | string | Entity DID |
| `name` | string? | Display name |
| `legalName` | string? | Legal name |
| `website` | string? | Website URL |
| `incorporationCountry` | string? | Country of incorporation |
| `isRegulated` | string? | Regulatory status |
| `verificationStatus` | string? | Verification status |
| `hasAdmin` | boolean? | Whether entity has an admin |
| `isNotabeneCustomer` | boolean? | Whether entity is a Notabene customer |
| `subsidiaries` | array? | Nested subsidiary entities |
| `badges` | string[]? | Entity badges |
| `isSandbox` | boolean? | Whether this is a sandbox entity |

Plus many more fields for address, travel rule protocol support, compliance phase, etc.

**Errors**: 400, 401, 403, 404, 409, 500

---

### `GET /network`

List entities in the network with filtering and pagination.

**Query Parameters:**

| Name | Type | Description |
|---|---|---|
| `q` | string | Search query |
| `emailDomain` | string | Filter by email domain |
| `badge` | string | `super_vasp`, `verified`, `in_network`, `claimed` |
| `jurisdictions` | string | Filter by jurisdictions |
| `page` | string | Page number |
| `per_page` | string | Results per page |
| `order` | string | Sort field (e.g. `name`) |
| `fields` | string | Comma-separated field list |
| `listingType` | string | `exclude_subsidiaries`, `all`, `exclude_gateways` |
| `listing` | string | `sandbox` or `non-sandbox` |
| `regulatoryStatus` | string | `Regulated`, `Not Regulated`, `Exempt`, `Pending` |
| `status` | string | `unreachable`, `receive`, `receive_respond`, `receive_respond_send` (pipe-separated for OR) |

**Response** `200`:

```json
{
  "vasps": [ { ... } ],
  "pagination": { "page": 0, "per_page": 100, "order": "name", "total": 100 }
}
```

---

### `GET /network/gleifSearch`

Search the GLEIF database for a Legal Entity Identifier (LEI).

**Query Parameters:**

| Name | Required | Type | Validation | Description |
|---|---|---|---|---|
| `leiNumber` | yes | string | Pattern: `^([A-Z0-9]){20}$` (exactly 20 alphanumeric chars) | LEI number to search |

**Response** `200`: `{ "results": [ { "provider": "gleif", "externalId", "entity": {…}, "registration": {…}, "updatedAt", "resolvedAt" } ] }`

---

## Relationships

### `POST /entities/{entityDID}/relationships`

Create a new relationship between an address and an entity. Optionally includes an ownership proof. The
relationship is confirmed immediately when your `entityDID` is its `from`, `to` or `custodian`.

**Request Body** (required):

```json
{
  "from": "did:pkh:eip155:1:0x...",
  "to": "did:web:counterparty.com",
  "custodian": "did:web:custodian.com",
  "proof": { ... }
}
```

| Field | Required | Validation | Description |
|---|---|---|---|
| `to` | yes | 3–255 chars | DID of the target entity |
| `from` | no | 3–255 chars; a `did:pkh:` value must hold a valid CAIP-10 address | DID of the source (address). Defaults to `entityDID` |
| `custodian` | no | valid DID, 3–255 chars | DID of the custodian |
| `proof` | no | | Ownership proof (see proof types below) |
| `asset` | no | | Asset used to normalize a plain `from` address to `did:pkh` |

**Proof types** (anyOf). Every proof also requires `status` (`pending`, `rejected`, `flagged`, `verified`), `did`
(valid DID) and `address` (CAIP-10; a `did:pkh:` prefix is stripped); `xpub` and `chainSpecificData` are optional.

1. **Cryptographic signature**: `type` is one of `eip-191`, `eip-712`, `eip-1271`, `bip-137`, `bip-322`, `tip-191`, `ed25519`, `xrp-ed25519`, `xlm-ed25519`, `cip-8`, `siwe`, `siwx`. Requires `proof` and `attestation`; optional `wallet_provider`.
2. **Screenshot**: `type: "screenshot"`, requires `url` (URI).
3. **Self-declaration**: `type: "self-declaration"`, requires `confirmed` (boolean); optional `attestation`.
4. **Micro-transfer**: `type: "microtransfer"`; optional `proof`, `chain`, `destination`, `amountSubunits`.

**Response** `201`:

```json
{
  "relationship": {
    "@id": "uuid",
    "from": "did:pkh:...",
    "to": "did:web:...",
    "custodian": null,
    "status": "UNCONFIRMED",
    "memoTag": null,
    "createdTime": "...",
    "updatedTime": "...",
    "proofId": "..."
  }
}
```

Status values: `UNCONFIRMED`, `CONFIRMED`, `PROVEN`

---

### `GET /entities/{entityDID}/relationships`

List all relationships for an entity.

**Query Parameters:**

| Name | Type | Description |
|---|---|---|
| `from` | string (did) | Filter by source DID |
| `to` | string (did) | Filter by target DID |
| `custodian` | string (did) | Filter by custodian DID |
| `asset` | string (CAIP-19) | Normalizes a plain `from` address to `did:pkh` |

**Response** `200`:

```json
{
  "relationships": [
    {
      "@id": "uuid",
      "from": "did:pkh:eip155:1:0x...",
      "to": "did:web:...",
      "custodian": "did:web:...",
      "status": "CONFIRMED",
      "confirmedBy": "did:web:...",
      "proofs": [
        {
          "status": "verified",
          "did": "did:pkh:...",
          "address": "eip155:1:0x742d35Cc...",
          "type": "eip-191",
          "proof": "0x...",
          "attestation": "...",
          "wallet_provider": "MetaMask"
        }
      ]
    }
  ]
}
```

Proof `status` and `type` values are those listed under POST.

---

### `PATCH /entities/{entityDID}/relationships`

Confirm a relationship. Creates it if it doesn't exist (upsert). Optionally includes ownership proof.

**Query Parameters**: `from`, `to`, `custodian`, `asset`. `from` and `to` are required, in the query or (for backwards
compatibility) in the body.

**Request Body** (optional): `{ "proof": { ... } }`, same proof structure as POST.

**Response** `201`: `{ "success": true }`

---

### `DELETE /entities/{entityDID}/relationships`

Delete an existing relationship.

**Query Parameters**: `from`, `to` (required), `asset` (optional)

**Response** `202`: `{ "success": true }`

---

## Transfers

### `POST /entities/{entityDID}/tx`

Create a new transfer. This is the entry point for initiating Travel Rule compliance.

**Request Body** (required):

```json
{
  "ref": "a7135b55-0822-4e96-86dc-3c43dfc6c333",
  "originator": { "@id": "mailto:user@example.com" },
  "beneficiary": { "@id": "mailto:recipient@example.com" },
  "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "amount": "100.00",
  "agents": [
    {
      "@id": "did:web:your-vasp.com",
      "role": "VASP",
      "for": "mailto:user@example.com",
      "policies": [
        { "@type": "REQUIRE_AUTHORIZATION" }
      ]
    },
    {
      "@id": "eip155:1:0xSettlementAddress",
      "role": "SettlementAddress"
    }
  ],
  "settlementId": "eip155:1:tx/0xabc123...",
  "transactionValue": { "amount": "100.00", "currency": "USD" },
  "blockchainAnalyticsAlias": "My Wallet",
  "memoTag": "12345"
}
```

| Field | Required | Type | Validation | Description |
|---|---|---|---|---|
| `ref` | yes | string | 1–255 chars, pattern: `^[a-zA-Z0-9_-]{1,255}$` | Unique reference for idempotency |
| `originator` | yes | object | `@id`: 3–255 chars, an IRI (`mailto:…`, `did:…`); a `did:` value must be a valid DID | `{ "@id": "..." }` |
| `beneficiary` | yes | object | same as `originator` | `{ "@id": "..." }` |
| `asset` | yes | string | CAIP-19, uppercase symbol of 2–20 chars (`[A-Z0-9-]`, e.g. `BTC`), or 10-char DTI | Asset identifier |
| `amount` | yes | string | Pattern: `^\d+(\.\d+)?$` (numeric, no negatives) | Decimal amount |
| `agents` | yes | array | minItems: 1 | Array of agents (see Agent schema) |
| `settlementId` | no | string | CAIP-220 (`eip155:1:tx/0x…`); raw hashes are normalized when the chain is known from the asset | On-chain transaction hash |
| `transactionValue` | no | object | `amount`: `^\d+(\.\d+)?$`; `currency`: 1–3 chars | `{ "amount": "...", "currency": "..." }` or null |
| `blockchainAnalyticsAlias` | no | string | | Alias for blockchain analytics |
| `memoTag` | no | string | 1–28 chars, pattern: `^[ -~]+$` (printable ASCII) | Memo/destination tag |

**Agent object:**

| Field | Required | Type | Description |
|---|---|---|---|
| `@id` | yes | string | 2–255 chars. A valid DID, except for `SourceAddress`/`SettlementAddress`, which also accept a plain or CAIP-10 address (normalized to `did:pkh`) |
| `role` | no | string | `VASP`, `Custodian`, `SettlementAddress`, `SourceAddress`, `Gateway`, `Unknown` (default) |
| `for` | no | string | 2–255 chars. Identifier of the party or agent this agent acts for |
| `policies` | no | array | minItems 1. `[{ "@type": "REQUIRE_AUTHORIZATION" | "REQUIRE_PRESENTATION" | "REQUIRE_RELATIONSHIP_CONFIRMATION" }]` |
| `memoTag` | no | string | 1–28 printable ASCII chars. Only allowed on `SourceAddress`/`SettlementAddress` agents |
| `selectedAsset` | no | string | Asset selected by this agent (CAIP-19 or ISO currency) |
| `selectedSettlementAddress` | no | string | Settlement address selected by this agent |

**Response** `201`: `{ "transfer": <Transfer object> }` (see [Transfer schema](#transfer)). The create response
carries only creation-time fields plus `requestForInformation` (`{ isRFI, direction, actionRequired }`); lifecycle
fields such as `isTravelRule`, `settlementAddress` and the `*Time` timestamps appear on GET.

**Errors**: 400, 401, 403, 404, 409 (`Transfer with ref … already exists`), 500

---

### `GET /entities/{entityDID}/tx`

List transfers for an entity with filtering and pagination.

**Query Parameters:**

| Name | Type | Default | Description |
|---|---|---|---|
| `limit` | integer | 25 | Min: 1, Max: 100 |
| `offset` | integer | 0 | Min: 0 |
| `agent` | array | - | Filter by agent DIDs (array syntax, `agent[]=…`) |
| `status` | array | - | Filter by transfer status (array syntax; values from the status list below) |
| `asset` | array | - | Filter by asset identifier (array syntax) |
| `party` | array | - | Filter by party identifiers (array syntax) |
| `refs` | string or array | - | Filter by reference identifiers |
| `created_before` | string | - | Date starting `YYYY-MM-DD` (time optional) |
| `created_after` | string | - | Date starting `YYYY-MM-DD` (time optional) |
| `direction` | string | - | `INCOMING` or `OUTGOING` |
| `search_term` | string | - | Full-text search |
| `fields` | string | - | Comma-separated fields to include |
| `sort_by` | string | - | Field to sort by |
| `sort_order` | string | - | `asc` or `desc` |
| `action_required` | boolean | - | Filter to transfers requiring action |
| `request_for_information` | boolean | - | Filter to transfers with pending info requests |

**Response** `200`:

```json
{
  "data": [ { ... } ],
  "pagination": { "total": 100, "limit": 25, "offset": 0 }
}
```

---

### `GET /entities/{entityDID}/tx/{transferId}`

Get a specific transfer with optional PII decryption and timeline.

**Path Parameters:** `transferId` (uuid)

**Query Parameters:**

| Name | Type | Default | Description |
|---|---|---|---|
| `decrypt` | string | `"false"` | `"true"` or `"false"` - decrypt PII data |
| `sanitize` | string | `"false"` | `"true"` or `"false"` - sanitize sensitive data |
| `include_timeline` | string | `"false"` | Include event timeline |
| `include_integration_results` | string | `"false"` | Include integration results |
| `include_replaced` | string | `"false"` | Include replaced agents in the response |
| `include_removed` | string | `"false"` | Include removed agents in the response |
| `include_comments` | string | `"false"` | Include comments |

**Response** `200`: `{ "transfer": <Transfer object> }`

---

### `GET /entities/{entityDID}/tx/search`

Search transfers with filters. Same query parameters as `GET /tx`. Returns `SearchTransfer` objects with `matchedColumns` field.

**Response** `200`:

```json
{
  "data": [
    {
      "id": "...",
      "ref": "...",
      "status": "...",
      "matchedColumns": ["ref", "settlementId"],
      ...
    }
  ],
  "pagination": { "total": 10, "limit": 25, "offset": 0 }
}
```

---

### `GET /entities/{entityDID}/tx/match`

Match transfers by settlement identifiers (e.g., on-chain transaction hash).

**Query Parameters:**

| Name | Required | Type | Description |
|---|---|---|---|
| `settlement_id` | yes | string | Tx hash, CAIP-220 (`eip155:1:tx/0x…`) or raw (normalized when the chain is known) |
| `settlement_address` | yes | string | Destination address: plain, CAIP-10 or `did:pkh` |
| `source_address` | no | string | Sender address (CAIP-10); an embedded memo tag is used for matching |
| `asset` | no | string | CAIP-19. Needed only when `settlement_address`, `source_address` and `settlement_id` are all plain (no chain) |
| `settlement_id_index` | no | string | Vout/log index (strict match first, dropped if broadened) |
| `memo_tag` | no | string | Memo tag for matching |
| `limit` | no | integer | Default: 25, Max: 250 |
| `offset` | no | integer | Default: 0 |
| `direction` | no | string | `INCOMING` or `OUTGOING` |
| `fields`, `sort_by`, `sort_order` | no | string | As for list |

**Response** `200`:

```json
{
  "data": [ { ... } ],
  "pagination": { "total": 1, "limit": 25, "offset": 0 },
  "meta": {
    "match_strategy": "strict",
    "broadened_candidates": 0
  }
}
```

`match_strategy`: `strict`, `broadened`, or `none`

---

### `POST /entities/{entityDID}/tx/{transferId}/authorize`

Authorize a transfer to proceed.

**Query Parameters:**

| Name | Type | Description |
|---|---|---|
| `force` | string | `"true"` to bypass certain checks |
| `ignoreFlags` | string | `"true"` to ignore flag policies |

**Request Body** (required; `{}` is valid):

```json
{
  "settlementAddress": "eip155:1:0x...",
  "memoTag": "12345"
}
```

| Field | Required | Validation | Description |
|---|---|---|---|
| `settlementAddress` | no | CAIP-10 (`eip155:1:0x…`) or RFC 8905 `payto://` URI | Settlement address (beneficiary side) |
| `memoTag` | no | string | Memo/destination tag |

Other body fields are ignored.

**Response** `202`: `{ "message": "Transfer authorization accepted" }`

---

### `POST /entities/{entityDID}/tx/{transferId}/reject`

Reject a transfer with a reason.

**Request Body** (required):

```json
{
  "reason": "COUNTERPARTY_RISK",
  "comment": "Optional comment (required when reason is OTHER)"
}
```

| Field | Required | Validation | Description |
|---|---|---|---|
| `reason` | yes | One of the enum values below | Rejection reason |
| `comment` | conditional | maxLength: 500. **Required** when reason is `OTHER` | Additional details |

**Reason values**: `COUNTERPARTY_RISK`, `COUNTERPARTY_DUE_DILIGENCE`, `BLOCKCHAIN_RISK_SCORE`, `SANCTION_SCREENING`, `ASSET_TYPE`, `SUSPICIOUS_TRANSACTION`, `COUNTERPARTY_POLICIES`, `COUNTERPARTY_REJECTED`, `COUNTERPARTY_NO_RESPONSE`, `CANCELED_BY_INITIATOR`, `REMOVED_FROM_TRANSFER`, `TRANSFER_PARTICIPANT`, `SOURCE_ADDRESS`, `BENEFICIARY_ADDRESS`, `BENEFICIARY_NOT_FOUND`, `ORIGINATOR_REJECT_OUTGOING`, `BENEFICIARY_REJECT_INCOMING`, `COMPLIANCE_POLICIES`, `OTHER`

**Response** `200`: `{ "message": "Transfer rejected successfully" }`

---

### `POST /entities/{entityDID}/tx/{transferId}/settle`

Settle a transfer by providing on-chain transaction details.

**Request Body** (required):

```json
{
  "settlementId": "eip155:1:tx/0xabc123...",
  "settlementIdIndex": "0",
  "revertSettlementId": "eip155:1:tx/0xdef456..."
}
```

All fields are optional. `settlementId` and `revertSettlementId`: max 255 chars, CAIP-220 recommended;
`settlementIdIndex`: max 64 chars (vout/log index).

**Response** `202`: `{ "message": "Transfer settlement accepted" }`

---

### `POST /entities/{entityDID}/tx/{transferId}/agents`

Add an agent to an existing transfer.

**Request Body** (required):

```json
{
  "agent": {
    "@id": "did:web:new-agent.com",
    "role": "VASP",
    "for": "mailto:user@example.com",
    "policies": [{ "@type": "REQUIRE_AUTHORIZATION" }]
  }
}
```

`agent` uses the [Agent object](#post-entitiesentitydidtx) schema.

**Response** `202`: `{ "message": "Agent addition accepted" }`

---

### `POST /entities/{entityDID}/tx/{transferId}/agents/replace`

Replace an existing agent with a new agent. Caller must be the transfer initiator, a gateway, or have a relationship with the agent being replaced.

**Request Body** (required):

```json
{
  "original": "did:web:old-agent.com",
  "replacement": {
    "@id": "did:web:new-agent.com",
    "role": "VASP",
    "for": "mailto:user@example.com"
  }
}
```

**Response** `202`: `{ "message": "Agent replacement accepted" }`

---

### `POST /entities/{entityDID}/tx/{transferId}/agents/remove`

Remove (soft-delete) an agent from a transfer. Same caller rule as replace.

**Request Body** (required): `{ "original": "did:web:agent.com", "reason": "optional, max 500 chars" }`

**Response** `202`: `{ "message": "Agent removal accepted" }`

---

### TAP Policies

#### `GET /entities/{entityDID}/tx/{transferId}/tap-policies`

List the TAP policies on all of the transfer's agents.

**Response** `200`:

```json
{
  "policies": [
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
  ]
}
```

#### `GET /entities/{entityDID}/tx/{transferId}/tap-policies/{policyId}`

Get one TAP policy, wrapped in the agent that owns it:

```json
{
  "agent": {
    "@id": "did:web:counterparty.com",
    "@type": "TransferAgent",
    "name": "Counterparty VASP",
    "for": "...",
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
for another entity.

**TapPolicy `@type` values**: `REQUIRE_AUTHORIZATION`, `REQUIRE_PRESENTATION`, `REQUIRE_RELATIONSHIP_CONFIRMATION`

**TapPolicy `status` values**: `COMPLETED`, `INCOMPLETE`, `PENDING`, `REPLACED`

---

## Transfer Checks

These self-checks exist on a transfer only when the entity has configured them (for beneficiaries with a
`REQUIRE_PRESENTATION` policy). The `tap.requirePresentationSatisfied` webhook carries a callback URL for each
configured check.

### `POST /entities/{entityDID}/tx/{transferId}/checks/beneficiary-name-matching`

Mark beneficiary name matching check as complete after verifying the beneficiary name matches your KYC records.

**Request Body**: None

**Response** `200`:

```json
{
  "id": "uuid",
  "transferId": "uuid",
  "entityDid": "did:web:...",
  "type": "REQUIRE_BENEFICIARY_NAME_MATCHING",
  "status": "COMPLETE",
  "policyData": {
    "confirmedBy": "did:web:...",
    "confirmedAt": "2025-01-15T10:00:00.000Z"
  },
  "createdAt": "...",
  "updatedAt": "..."
}
```

---

### `POST /entities/{entityDID}/tx/{transferId}/checks/internal-checks`

Mark internal checks as complete after completing operational checks on PII/Travel Rule information.

**Request Body**: None

**Response** `200`: Same schema as beneficiary-name-matching with `type: "REQUIRE_INTERNAL_CHECKS"`.

---

## PII

### `POST /entities/{entityDID}/tx/{transferId}/append`

Append PII (IVMS101 data) to a transfer for Travel Rule compliance (Notabene-managed encryption). The PII is
validated against the transfer's presentation definition, if set, and encrypted before storage.

**Request Body** (required):

```json
{
  "originator": { "@id": "mailto:originator@notabene.id" },
  "beneficiary": { "@id": "mailto:beneficiary@notabene.id" },
  "ivms101": {
    "originator": {
      "originatorPerson": [{
        "naturalPerson": {
          "name": {
            "nameIdentifier": [{
              "primaryIdentifier": "Doe",
              "secondaryIdentifier": "John",
              "naturalPersonNameIdentifierType": "LEGL"
            }]
          },
          "dateAndPlaceOfBirth": {
            "dateOfBirth": "1980-01-01",
            "placeOfBirth": "Anytown, US"
          },
          "geographicAddress": [{
            "addressType": "HOME",
            "addressLine": ["123 Main Street"],
            "townName": "Anytown",
            "country": "US",
            "postCode": "1234"
          }]
        }
      }],
      "accountNumber": ["originator-account"]
    },
    "beneficiary": {
      "beneficiaryPerson": [{
        "naturalPerson": {
          "name": {
            "nameIdentifier": [{
              "primaryIdentifier": "Smith",
              "secondaryIdentifier": "Jane",
              "naturalPersonNameIdentifierType": "LEGL"
            }]
          }
        }
      }],
      "accountNumber": ["beneficiary-account"]
    }
  }
}
```

| Field | Required | Description |
|---|---|---|
| `ivms101` | yes | IVMS101 data (see [IVMS101 schema](#ivms101)) |
| `originator`, `beneficiary` | no | `{ "@id": "..." }`. Giving both enables PII reuse |
| `agents` | no | Agents to add during the append (max 25) |

**Response** `202`: `{ "message": "Transfer appended successfully with encrypted PII" }`

---

### `POST /entities/{entityDID}/tx/{transferId}/policies/{policyId}/presentation`

Present PII data to counterparties for a specific policy.

**Path Parameters**: `transferId`, `policyId`

**Query Parameters**: `skipValidation` (`"true"` skips validation; default `false`)

**Request Body** (required): `{ "ivms101": <IVMS101 object> }`

**Response** `202`: `{ "message": "Presentation accepted and processing" }` (a retry of a completed policy also
answers `202`)

`POST /entities/{entityDID}/tx/{transferId}/presentation` (no `policyId`) presents to every `REQUIRE_PRESENTATION`
policy on the transfer. Use these routes when the webhook's `callbackUrl` points at them (entity-managed encryption);
a policy that is `for` your own entity must be fulfilled with `append` instead.

---

## Schemas

### Transfer

The core transfer object returned by create, get, and list endpoints.

| Field | Type | Required | Description |
|---|---|---|---|
| `@id` | string | yes | Unique transfer identifier |
| `id` | string | no | Alternate ID |
| `status` | string | yes | See status values below |
| `asset` | string? | no | CAIP-19 asset identifier |
| `initiator` | string | yes | DID of the initiating entity |
| `agents` | array | yes | Transfer agents. Each has `@id` (the TransferAgent's own id), `agent: { "@id": <agent DID or address> }`, `role`, `for`, `status`, `policies`, `relationship`, and optional `name`, `memoTag`, `replacement` |
| `amount` | string? | no | Transfer amount |
| `originator` | object? | no | `{ "@id": "...", "originatorPerson": [...] }` |
| `beneficiary` | object? | no | `{ "@id": "...", "beneficiaryPerson": [...] }` |
| `ref` | string? | no | Client reference |
| `direction` | string? | no | `INCOMING` or `OUTGOING` |
| `transactionType` | string? | no | `TRANSFER`, `PAYOUT`, or `PAYIN` |
| `settlementAddress` | string? | no | Settlement address (CAIP-10) |
| `settlementId` | string? | no | On-chain transaction hash (CAIP-220) |
| `settlementIdIndex` | string? | no | Vout/log index |
| `isTravelRule` | boolean? | no | Whether travel rule applies |
| `internalAuthorization` | string? | no | Your own decision: `PENDING`, `AUTHORIZED`, `REJECTED` |
| `authorizationRequested` | boolean? | no | Whether authorization has been requested |
| `memoTag` | string? | no | Memo/destination tag |
| `createdTime` | string? | no | Creation timestamp |
| `authorizedTime` | string? | no | When authorized |
| `rejectedTime` | string? | no | When rejected |
| `settledTime` | string? | no | When settled |
| `revertedTime` | string? | no | When reverted |
| `expiresTime` | string? | no | Expiration time |
| `amounts` | array? | no | Detailed amounts with currency conversions |
| `flags` | array? | no | Transfer flags |
| `log` | array? | no | Activity log |
| `policyEvaluations` | array? | no | Policy evaluation results |
| `createdAt`, `updatedAt` | string? | no | Record timestamps |
| `originator.encryptedIvms101`, `beneficiary.encryptedIvms101` | string? | no | Compact JWE of the party's IVMS101, present only when it was sealed to a key under your DID that the node does not hold |

**Transfer status values**: `AWAITING-YOURS` (waiting on you), `AWAITING-COUNTERPARTY` (waiting on the counterparty), `REJECTED`, `AUTHORIZED`, `FLAGGED`, `SETTLED`, `FLAGGED-SETTLEMENT`, `REVERTED`, `REVERT-AUTHORIZED`, `REVERT-REJECTED`, `REVERT-FLAGGED`, `FROZEN`, `CLEARED`, `REVERT-REQUESTED`. `OUTGOING` and `INCOMING` are deprecated (still found on older transfers); new transitions emit the `AWAITING-*` values.

**Agent status values**: `PROCESSING`, `AUTHORIZED`, `REJECTED`, `SETTLED`, `FAILED`, `REPLACED`, `UNREACHABLE`

**Agent role values**: `VASP`, `Custodian`, `SettlementAddress`, `SourceAddress`, `Gateway`, `Unknown`

---

### IVMS101

IVMS 101 data for originator and beneficiary PII (v2 field names: `originatorPerson`, `beneficiaryPerson`).

The API schema makes every field optional but rejects unknown keys, caps every array at 25 items, and allows a person
to be a `naturalPerson` or a `legalPerson`, not both. Which fields must be present is decided by the transfer's
presentation definition (the counterparty's jurisdiction); `(IVMS)` marks fields IVMS101 itself requires when their
parent is present.

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
    accountNumber[]: string, max 255
  beneficiary:
    beneficiaryPerson[]: (same structure as originatorPerson)
    accountNumber[]: string, max 255
```
