# Notabene Webhooks Documentation

This document provides comprehensive information about all webhook events sent by the Notabene platform to receiving developers. Webhooks are delivered via [Svix](https://svix.com/) with automatic retries, exponential backoff, and signature verification.

## Table of Contents

- [Overview](#overview)
- [Webhook Envelope Structure](#webhook-envelope-structure)
- [Webhook Categories](#webhook-categories)
- [Setup](#setup)
- [Authentication & Security](#authentication--security)
- [Webhook Events](#webhook-events)
  - [Transfer Management Events](#transfer-management-events)
  - [TAP Policy Requirement Events](#tap-policy-requirement-events)
  - [TAP Policy Satisfaction Events](#tap-policy-satisfaction-events)
  - [TAP Policy Status Events](#tap-policy-status-events)
  - [Settlement Signal Event](#settlement-signal-event)
- [Common Fields](#common-fields)
- [Error Handling](#error-handling)
- [Testing](#testing)

## Overview

Notabene webhooks notify your application about important events in real-time during the transaction authorization process. These events are critical for maintaining compliance with travel rule requirements and managing the transfer lifecycle.

**Delivery Method**: HTTP POST to your configured endpoint URLs
**Delivery Service**: Svix (reliable delivery with retries)
**Content-Type**: `application/json`
**Signature Verification**: All webhooks are signed for security

## Webhook Envelope Structure

All webhook events are delivered inside a standard envelope. The `message` field contains the event type string, the `payload` field contains the event-specific data, and the `version` field indicates the payload format version.

```json
{
  "message": "notification.transferCreated",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123"
  },
  "version": "1.0.0"
}
```

| Field     | Type   | Description                                      |
| --------- | ------ | ------------------------------------------------ |
| `message` | string | The webhook event type identifier                |
| `payload` | object | Event-specific data (see individual event docs)  |
| `version` | string | Payload format version (currently `"1.0.0"`)     |

> **Note**: The event type is only present in the `message` field of the envelope. It is **not** included inside the `payload` object.

## Webhook Categories

### 1. Transfer Management Events

These webhooks notify you about changes in transfer state and agent management:

- **notification.transferCreated**: A new transfer has been initiated
- **notification.transferStatusChanged**: Transfer status has been updated
- **notification.transferAgentAdded**: A new agent has been added to a transfer
- **notification.transferAgentReplaced**: An agent has been replaced in a transfer
- **notification.transferAgentRemoved**: An agent has been removed from a transfer
- **notification.transferAgentStatusChanged**: An agent's status has been updated

### 2. TAP Policy Requirement Events

These webhooks are sent when the Notabene system requires external actions to continue processing a transfer:

- **tap.requirePresentationRequested**: Request for credential/document presentation
- **tap.requireAuthorizationRequested**: Request for transfer authorization decision
- **tap.requireRelationshipConfirmationRequested**: Request to confirm entity relationships

### 3. TAP Policy Satisfaction Events

These webhooks confirm that previously requested actions have been completed:

- **tap.requireAuthorizationSatisfied**: Authorization requirement has been fulfilled
- **tap.requirePresentationSatisfied**: Presentation requirement has been fulfilled
- **tap.requireRelationshipConfirmationSatisfied**: Relationship confirmation requirement has been fulfilled

### 4. TAP Policy Status Events

These webhooks provide updates on the overall status of policy processing:

- **tap.requirePresentationPartiallySatisfied**: A presentation requirement has been partially fulfilled (some but not all requested credentials received)

### 5. Settlement Signal

- **notification.settleTransaction**: Settlement-readiness signal (only for entities with a settlement policy and settlement signals enabled by Notabene)

Other event types exist for Flow payments (`flow.*`) and DDQ connections (`connection.*`); they are outside this guide.

## Setup

Webhooks are only delivered once the entity has a Svix application:

- `PUT /entities/{entityDID}/webhook` — enables webhooks (creates the Svix application; idempotent). Returns `{ "svix": <application> }`.
- `GET /entities/{entityDID}/webhook?generate_portal_url=true` — returns `{ "svix": <application>, "svixPortalUrl": "..." }`. Add endpoint URLs, pick event types and copy the endpoint signing secret (`whsec_…`) in that portal.
- `DELETE /entities/{entityDID}/webhook` — disables webhooks.

## Authentication & Security

All webhooks are signed by Svix (`svix-id`, `svix-timestamp`, `svix-signature` headers). Verify them against the **raw** request body with the endpoint's signing secret:

```javascript
import { Webhook } from "svix";

const webhook = new Webhook(process.env.SVIX_WEBHOOK_SECRET);

// Verify webhook signature
try {
  const payload = webhook.verify(body, headers);
  // Process the webhook...
} catch (err) {
  // Invalid signature
  console.error("Webhook signature verification failed");
}
```

---

## Webhook Events

---

### Transfer Management Events

#### notification.transferCreated

**Purpose**: Notifies that a new transfer has been created in the system.

**When Sent**: Immediately after a transfer is successfully created and enters the processing pipeline.

**Expected Action**: Optional - log the event, update internal records, or trigger business workflows.

**Payload Structure**:

```json
{
  "message": "notification.transferCreated",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field | Type   | Required | Description                       |
| ----- | ------ | -------- | --------------------------------- |
| `for` | string | Yes      | Entity DID receiving this webhook |
| `id`  | string | Yes      | Transfer ID                       |

**Response Required**: None (informational only).

---

#### notification.transferStatusChanged

**Purpose**: Notifies that a transfer's status has changed.

**When Sent**: When a transfer moves between states (e.g., from `AWAITING-COUNTERPARTY` to `AUTHORIZED`, `AUTHORIZED` to `SETTLED`).

**Expected Action**: Update your system's transfer status tracking.

**Payload Structure**:

```json
{
  "message": "notification.transferStatusChanged",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "fromStatus": "AWAITING-COUNTERPARTY",
    "toStatus": "AUTHORIZED",
    "settlementId": "eip155:1:tx/0xabc..."
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field          | Type   | Required | Description                                  |
| -------------- | ------ | -------- | -------------------------------------------- |
| `for`          | string | Yes      | Entity DID receiving this webhook            |
| `id`           | string | Yes      | Transfer ID                                  |
| `fromStatus`   | string | Yes      | Previous transfer status                     |
| `toStatus`     | string | Yes      | New transfer status                          |
| `settlementId` | string | No       | Settlement identifier (when status involves settlement) |

**Response Required**: None (informational only).

**Status Values**: `AWAITING-YOURS` (new transfers start here), `AWAITING-COUNTERPARTY`, `AUTHORIZED`, `REJECTED`, `SETTLED`, `FLAGGED`, `FLAGGED-SETTLEMENT`, `REVERTED`, `REVERT-REQUESTED`, `REVERT-AUTHORIZED`, `REVERT-REJECTED`, `REVERT-FLAGGED`, `FROZEN`, `CLEARED`. Legacy transfers may still show the deprecated `OUTGOING` / `INCOMING`. There is no `PENDING` transfer status.

---

#### notification.transferAgentAdded

**Purpose**: Notifies that a new agent has been added to a transfer.

**When Sent**: When an intermediary entity or service provider is added to facilitate the transfer.

**Expected Action**: Update transfer tracking to include the new agent.

**Payload Structure**:

```json
{
  "message": "notification.transferAgentAdded",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "agentID": "agent-789",
    "agent": {
      "id": "agent-789",
      "for": "did:web:intermediary.com",
      "role": "VASP"
    }
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field        | Type   | Required | Description                       |
| ------------ | ------ | -------- | --------------------------------- |
| `for`        | string | Yes      | Entity DID receiving this webhook |
| `id`         | string | Yes      | Transfer ID                       |
| `agentID`    | string | Yes      | ID of the added agent             |
| `agent`      | object | Yes      | Agent details                     |
| `agent.id`   | string | No       | Agent identifier                  |
| `agent.for`  | string | No       | Entity DID the agent acts for     |
| `agent.role` | string | No       | Agent role                        |

**Response Required**: None (informational only).

**Agent Roles**: `VASP`, `Custodian`, `SettlementAddress`, `SourceAddress`, `Gateway`, `Unknown`

---

#### notification.transferAgentReplaced

**Purpose**: Notifies that an agent in a transfer has been replaced with a different agent.

**When Sent**: When an intermediary or service provider is changed during transfer processing.

**Expected Action**: Update your records to reflect the agent change.

**Payload Structure**:

```json
{
  "message": "notification.transferAgentReplaced",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "fromAgentID": "old-agent-123",
    "toAgentID": "new-agent-456",
    "toAgent": {
      "id": "new-agent-456",
      "for": "did:web:new-intermediary.com",
      "role": "VASP"
    }
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field          | Type   | Required | Description                         |
| -------------- | ------ | -------- | ----------------------------------- |
| `for`          | string | Yes      | Entity DID receiving this webhook   |
| `id`           | string | Yes      | Transfer ID                         |
| `fromAgentID`  | string | Yes      | ID of the agent being replaced      |
| `toAgentID`    | string | Yes      | ID of the new agent                 |
| `toAgent`      | object | Yes      | New agent details                   |
| `toAgent.id`   | string | No       | Agent identifier                    |
| `toAgent.for`  | string | No       | Entity DID the agent acts for       |
| `toAgent.role` | string | No       | Agent role                          |

**Response Required**: None (informational only).

---

#### notification.transferAgentRemoved

**Purpose**: Notifies that an agent has been removed from a transfer.

**Payload Structure**:

```json
{
  "message": "notification.transferAgentRemoved",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "removedAgentID": "agent-789",
    "reason": "..."
  },
  "version": "1.0.0"
}
```

**Payload Fields**: `for` (string, required), `id` (transfer ID, required), `removedAgentID` (string, required), `reason` (string, optional).

**Response Required**: None (informational only).

---

#### notification.transferAgentStatusChanged

**Purpose**: Notifies that an agent's status within a transfer has changed.

**When Sent**: When an agent's processing state changes (e.g., from `PROCESSING` to `AUTHORIZED`).

**Expected Action**: Update agent status tracking.

**Payload Structure**:

```json
{
  "message": "notification.transferAgentStatusChanged",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "agentID": "agent-456",
    "fromStatus": "PROCESSING",
    "toStatus": "AUTHORIZED"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field        | Type   | Required | Description                       |
| ------------ | ------ | -------- | --------------------------------- |
| `for`        | string | Yes      | Entity DID receiving this webhook |
| `id`         | string | Yes      | Transfer ID                       |
| `agentID`    | string | Yes      | Agent identifier                  |
| `fromStatus` | string | Yes      | Previous agent status             |
| `toStatus`   | string | Yes      | New agent status                  |

**Response Required**: None (informational only).

**Agent Statuses**: `PROCESSING`, `AUTHORIZED`, `REJECTED`, `SETTLED`, `FAILED`, `REPLACED`, `UNREACHABLE`

---

### TAP Policy Requirement Events

#### tap.requirePresentationRequested

**Purpose**: Requests the recipient to provide required credentials or documents for travel rule compliance.

**When Sent**: When a transfer requires additional documentation (identity verification, transaction details, etc.) to proceed.

**Expected Action**: Your system should facilitate the presentation of the required credentials to continue the transfer process.

**Payload Structure**:

```json
{
  "message": "tap.requirePresentationRequested",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "originatorAgentID": "originator-agent-123",
    "beneficiaryAgentID": "beneficiary-agent-456",
    "originatorPresentationDefinition": "...",
    "beneficiaryPresentationDefinition": "...",
    "callbackUrl": "/entities/did:web:example.com/transfers/transfer-123/append",
    "encryptionKey": "...",
    "policyId": "policy-789"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field                                | Type   | Required | Description                                                        |
| ------------------------------------ | ------ | -------- | ------------------------------------------------------------------ |
| `for`                                | string | Yes      | Entity DID receiving this webhook                                  |
| `id`                                 | string | Yes      | Transfer ID                                                        |
| `originatorAgentID`                  | string | Yes      | Originator agent identifier                                        |
| `beneficiaryAgentID`                 | string | No       | Beneficiary agent identifier (included when beneficiary presentation is required) |
| `originatorPresentationDefinition`   | string | Yes      | Presentation definition for originator credentials (may be empty)  |
| `beneficiaryPresentationDefinition`  | string | No       | Presentation definition for beneficiary credentials (only present when beneficiary data is required) |
| `callbackUrl`                        | string | Yes      | Relative path to POST PII to: `/entities/{did}/transfers/{id}/append` (Notabene-managed encryption) or `/entities/{did}/transfers/{id}/policies/{policyId}/presentation` (entity-managed encryption) |
| `encryptionKey`                      | object | No       | Encryption key for securing presentation data                      |
| `policyId`                           | string | Yes      | Policy identifier                                                  |

**Response Required**: POST the IVMS101 data to `callbackUrl` (see the append / presentation endpoints in the API reference) to satisfy the requirement.

---

#### tap.requireAuthorizationRequested

**Purpose**: Requests authorization decision for a transfer (approve or reject).

**When Sent**: When a transfer requires manual approval due to compliance policies, risk factors, or business rules.

**Expected Action**: Review the transfer details and provide an authorization decision (approve/reject).

**Payload Structure**:

```json
{
  "message": "tap.requireAuthorizationRequested",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "originatorAgentID": "originator-agent-123",
    "beneficiaryAgentID": "beneficiary-agent-456",
    "authorizeCallbackUrl": "/entities/did:web:example.com/transfers/transfer-123/authorize",
    "rejectCallbackUrl": "/entities/did:web:example.com/transfers/transfer-123/reject",
    "purpose": "High-value transfer requires manual review"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field                  | Type   | Required | Description                                      |
| ---------------------- | ------ | -------- | ------------------------------------------------ |
| `for`                  | string | Yes      | Entity DID receiving this webhook                |
| `id`                   | string | Yes      | Transfer ID                                      |
| `originatorAgentID`    | string | Yes      | Originator agent identifier                      |
| `beneficiaryAgentID`   | string | Yes      | Beneficiary agent identifier                     |
| `authorizeCallbackUrl` | string | Yes      | Relative URL path to POST approval               |
| `rejectCallbackUrl`    | string | Yes      | Relative URL path to POST rejection              |
| `purpose`              | string | No       | Reason the authorization is required             |

**Response Required**: POST to either `authorizeCallbackUrl` (to approve) or `rejectCallbackUrl` (to reject) the transfer.

> **Note**: The callback URLs are relative paths with the entity DID not URL-encoded. Prepend the API base URL (`https://api.eu1.notabene.id`) and URL-encode the DID segment. The bodies are those of the authorize / reject endpoints (reject requires a `reason`).

---

#### tap.requireRelationshipConfirmationRequested

**Purpose**: Requests confirmation of the relationship between entities in a transfer.

**When Sent**: When the system needs to verify that the relationship between originator and beneficiary entities is legitimate.

**Expected Action**: Confirm the relationship exists and is valid for this transfer.

**Payload Structure**:

```json
{
  "message": "tap.requireRelationshipConfirmationRequested",
  "payload": {
    "for": "did:web:example.com",
    "from": "did:pkh:eip155:1:0x1234...",
    "id": "transfer-123",
    "originatorAgentID": "originator-agent-123",
    "beneficiaryAgentID": "beneficiary-agent-456",
    "confirmCallbackUrl": "/entities/did:web:example.com/relationships?from=did:pkh:eip155:1:0x1234...&to=did:web:example.com",
    "purpose": "Confirm relationship between entities"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field                | Type   | Required | Description                                    |
| -------------------- | ------ | -------- | ---------------------------------------------- |
| `for`                | string | Yes      | Entity DID receiving this webhook              |
| `from`               | string | Yes      | Agent (usually the address DID) whose relationship must be confirmed |
| `id`                 | string | Yes      | Transfer ID                                    |
| `originatorAgentID`  | string | Yes      | Originator agent identifier                    |
| `beneficiaryAgentID` | string | Yes      | Beneficiary agent identifier                   |
| `confirmCallbackUrl` | string | Yes      | Relative path (with `from`/`to` query) to `PATCH` to confirm |
| `purpose`            | string | No       | Reason the confirmation is required            |

**Response Required**: `PATCH` `confirmCallbackUrl` (the confirm-relationship endpoint, `PATCH /entities/{entityDID}/relationships?from=…&to=…`).

> **Note**: The callback URL is a relative path with unencoded DIDs. Prepend the API base URL and URL-encode the path DID and query values.

---

### TAP Policy Satisfaction Events

#### tap.requireAuthorizationSatisfied

**Purpose**: Confirms that a previously requested authorization has been completed.

**When Sent**: After an authorization decision has been made and processed (following a tap.requireAuthorizationRequested event).

**Expected Action**: Optional - log the completion, update audit trails.

**Payload Structure**:

```json
{
  "message": "tap.requireAuthorizationSatisfied",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "agentID": "agent-456"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field     | Type   | Required | Description                       |
| --------- | ------ | -------- | --------------------------------- |
| `for`     | string | Yes      | Entity DID receiving this webhook |
| `id`      | string | Yes      | Transfer ID                       |
| `agentID` | string | Yes      | Agent that satisfied the requirement |

**Response Required**: None (informational only).

---

#### tap.requirePresentationSatisfied

**Purpose**: Confirms that a previously requested presentation has been completed. Sent to both parties. Includes callback URLs for the beneficiary self-policy checks (beneficiary name matching and internal checks) when those checks are configured.

**When Sent**: After required credentials/documents have been successfully presented and PII validation has passed (following a tap.requirePresentationRequested event).

**Expected Action**: Use the provided callback URLs to confirm beneficiary name matching and internal checks.

**Payload Structure**:

```json
{
  "message": "tap.requirePresentationSatisfied",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "agentID": "agent-456",
    "policyType": "REQUIRE_PRESENTATION",
    "confirmBeneficiaryCallbackUrl": "/entities/did%3Aweb%3Aexample.com/tx/transfer-123/checks/beneficiary-name-matching",
    "confirmInternalChecksCallbackUrl": "/entities/did%3Aweb%3Aexample.com/tx/transfer-123/checks/internal-checks"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field                              | Type   | Required | Description                                         |
| ---------------------------------- | ------ | -------- | --------------------------------------------------- |
| `for`                              | string | Yes      | Entity DID receiving this webhook                   |
| `id`                               | string | Yes      | Transfer ID                                         |
| `agentID`                          | string | Yes      | Agent that satisfied the requirement                |
| `policyType`                       | string | Yes      | `REQUIRE_PRESENTATION`, or `COUNTERPARTY_REQUIRE_PRESENTATION` on the copy sent to the counterparty |
| `confirmBeneficiaryCallbackUrl`    | string | No       | Relative URL path to confirm beneficiary name matching (only when that check is configured) |
| `confirmInternalChecksCallbackUrl` | string | No       | Relative URL path to confirm internal checks (only when that check is configured) |

**Response Required**: POST to each callback URL present to complete the self-policy checks. Pending checks must be completed before the transfer can proceed to authorization.

> **Note**: The callback URLs are relative paths. Prepend the Notabene API base URL to form the full endpoint.

---

#### tap.requireRelationshipConfirmationSatisfied

**Purpose**: Confirms that a previously requested relationship confirmation has been completed.

**When Sent**: After entity relationships have been successfully confirmed (following a tap.requireRelationshipConfirmationRequested event).

**Expected Action**: Optional - log the completion, update relationship records.

**Payload Structure**:

```json
{
  "message": "tap.requireRelationshipConfirmationSatisfied",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "relationshipId": "relationship-789",
    "agentID": "did:pkh:eip155:1:0x1234...",
    "from": "did:pkh:eip155:1:0x1234...",
    "to": "did:web:example.com",
    "status": "CONFIRMED"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field            | Type   | Required | Description                       |
| ---------------- | ------ | -------- | --------------------------------- |
| `for`            | string | Yes      | Entity DID receiving this webhook |
| `id`             | string | Yes      | Transfer ID                       |
| `relationshipId` | string | Yes      | Relationship identifier           |
| `agentID`        | string | Yes      | Same as `from`                    |
| `from`           | string | Yes      | Relationship source (address DID) |
| `to`             | string | Yes      | Relationship target (entity DID)  |
| `status`         | string | Yes      | Relationship status (`CONFIRMED` or `PROVEN`) |

**Response Required**: None (informational only).

---

### TAP Policy Status Events

#### tap.requirePresentationPartiallySatisfied

**Purpose**: Notifies that a presentation requirement has been partially satisfied. Some but not all of the requested credentials have been received.

**When Sent**: When a subset of the requested credentials/documents has been presented, but additional credentials are still needed to fully satisfy the requirement.

**Expected Action**: Optional - track presentation progress. The system will continue to collect remaining credentials. A `tap.requirePresentationSatisfied` event will follow once all credentials are received.

**Payload Structure**:

```json
{
  "message": "tap.requirePresentationPartiallySatisfied",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "policyId": "policy-789"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field       | Type   | Required | Description                                     |
| ----------- | ------ | -------- | ----------------------------------------------- |
| `for`       | string | Yes      | Entity DID receiving this webhook               |
| `id`        | string | Yes      | Transfer ID                                     |
| `policyId`  | string | Yes      | UUID of the policy being partially satisfied    |

**Response Required**: None (informational only).

---

### Settlement Signal Event

#### notification.settleTransaction

**Purpose**: Tells the entity a transfer is ready to settle, or needs action. Only sent when Notabene has enabled settlement signals and the entity has a settlement policy.

**Payload Fields** (besides `for` and `id`): `transferId`, `trigger` (`INTERNAL_APPROVED` | `AUTHORIZATION_COMPLETE` | `COUNTERPARTY_REJECTED` | `TIMEOUT`), `action` (`SETTLE` | `REVIEW` | `CANCEL` | `ACK`), `transferStatus`, `settlementReadiness`, `internalPoliciesComplete`, `authorizationRequirementsComplete`, `authorizationRequirementsPending[]` (`requirement`, `from`, `responded`, `timedOut`, `timeoutAt?`), `timedOut`, `timeoutAction?` (`AUTHORIZE` | `FLAG` | `REJECT`), `counterpartyRejectedAction?` (`FLAG` | `REJECT`).

---

## Common Fields

All webhook `payload` objects include these standard fields:

- **`for`**: String DID of the entity this webhook is intended for
- **`id`**: String transfer ID
- **`agentID`** (when applicable): String identifier of the agent involved
- **Timestamp fields**: Automatically added by Svix to the HTTP headers (e.g., `svix-timestamp`, `svix-id`, etc.)

The webhook envelope always includes:

- **`message`**: The event type string (e.g., `notification.transferCreated`)
- **`payload`**: The event-specific data
- **`version`**: Currently `"1.0.0"`

## Error Handling

### Webhook Delivery Failures

Svix automatically handles webhook delivery failures with:

- **Exponential backoff**: Increasing delays between retry attempts, per Svix's retry schedule
- **Manual replay**: Failed messages can be inspected and resent from the Svix portal

### Recommended Error Handling

```javascript
app.post("/notabene-webhook", async (req, res) => {
  try {
    // Verify signature first (req.body must be the raw body, e.g. express.raw())
    const verified = webhook.verify(req.body, req.headers);

    // The verified object has the envelope structure
    const { message, payload, version } = verified;

    // Process the webhook based on the message (event type)
    await processWebhook(message, payload);

    // Return success
    res.status(200).send("OK");
  } catch (error) {
    console.error("Webhook processing failed:", error);

    // Return error status to trigger Svix retry
    res.status(500).send("Processing failed");
  }
});
```

### HTTP Response Codes

- **200-299**: Success - webhook processed successfully
- **Any other status or timeout**: Treated as a failure and retried

## Testing

### Webhook Testing Tools

1. **Svix Dashboard**: View webhook delivery status and payloads
2. **ngrok**: Create public tunnels for local development
3. **webhook.site**: Simple webhook testing service
4. **Postman/Insomnia**: Test webhook endpoints manually

### Example Test Setup

```javascript
// Test webhook endpoint
app.post("/test-webhook", (req, res) => {
  console.log("Received webhook:", JSON.stringify(req.body, null, 2));
  console.log("Headers:", req.headers);
  res.status(200).send("Received");
});
```

### Development Tips

1. **Always verify signatures** in production
2. **Handle idempotency** - webhooks may be delivered multiple times
3. **Log webhook payloads** for debugging and audit purposes
4. **Return 200 status** as quickly as possible to avoid timeouts
5. **Process webhooks asynchronously** if they trigger complex operations
6. **Monitor webhook failures** in the Svix dashboard

## Integration Examples

### Node.js Express Example

```javascript
import express from "express";
import { Webhook } from "svix";

const app = express();
const webhook = new Webhook(process.env.SVIX_WEBHOOK_SECRET);

app.use("/webhook", express.raw({ type: "application/json" }));

app.post("/webhook", async (req, res) => {
  const rawPayload = req.body;
  const headers = req.headers;

  try {
    // Verify the webhook signature
    const event = webhook.verify(rawPayload, headers);

    // event has the envelope structure: { message, payload, version }
    const { message, payload } = event;

    // Handle different event types using the `message` field
    switch (message) {
      case "tap.requireAuthorizationRequested":
        await handleAuthorizationRequest(payload);
        break;
      case "notification.transferStatusChanged":
        await handleStatusChange(payload);
        break;
      // ... handle other event types
    }

    res.status(200).send("OK");
  } catch (err) {
    console.error("Webhook signature verification failed:", err);
    res.status(400).send("Invalid signature");
  }
});

async function handleAuthorizationRequest(payload) {
  // Implement your authorization logic
  console.log(`Authorization required for transfer ${payload.id}`);

  // Make decision and callback to Notabene
  const decision = await makeAuthorizationDecision(payload);

  // Callback paths are relative and carry the raw entity DID: encode it
  const baseUrl = "https://api.eu1.notabene.id";
  const encode = (path) => path.replace(payload.for, encodeURIComponent(payload.for));
  const headers = {
    "Content-Type": "application/json",
    Authorization: `Bearer ${accessToken}`,
  };

  if (decision.approved) {
    await fetch(`${baseUrl}${encode(payload.authorizeCallbackUrl)}`, {
      method: "POST",
      headers,
      body: JSON.stringify({}),
    });
  } else {
    await fetch(`${baseUrl}${encode(payload.rejectCallbackUrl)}`, {
      method: "POST",
      headers,
      body: JSON.stringify({ reason: "COMPLIANCE_POLICIES", comment: decision.reason }),
    });
  }
}
```

## Support

For webhook integration support:

- **Documentation**: Review this guide and the Svix documentation
- **Dashboard**: Use the Svix dashboard to monitor webhook delivery
- **Support**: Contact Notabene support for integration assistance
- **Status**: Check webhook delivery status in the Svix interface

---

_This documentation covers the transfer-related webhook events of the v2 API. Webhook payloads and event types may be extended in future versions._
