# Notabene Webhook Events (v2)

Catalogue of every webhook event. Mechanics live in the main guide:

- Setup, Svix signature verification: [guide-travel-rule-api.md#webhooks](guide-travel-rule-api.md#webhooks)
- Envelope `{ message, payload, version }` (event type in `message`, transfer id in `payload.id`):
  [guide-travel-rule-api.md#webhook-envelope](guide-travel-rule-api.md#webhook-envelope)
- At-least-once, out-of-order delivery and idempotent handlers:
  [guide-travel-rule-api.md#delivery-guarantees](guide-travel-rule-api.md#delivery-guarantees)

Every `payload` carries `for` (DID of the entity the webhook is for) and `id` (transfer id); the tables list the other
fields. Callback URLs are relative paths: prepend the API base URL and call them as given (some still use the
deprecated `/transfers/...` prefix). Respond `2xx` once processed; `4xx` is not retried, `5xx` is retried.

## Table of Contents

- [Transfer Management Events](#transfer-management-events)
- [TAP Policy Requirement Events](#tap-policy-requirement-events)
- [TAP Policy Satisfaction Events](#tap-policy-satisfaction-events)
- [TAP Policy Status Events](#tap-policy-status-events)

---

## Transfer Management Events

| `message` | Fires when | Key payload fields | What to do |
|---|---|---|---|
| `notification.transferCreated` | A transfer is created and enters processing | — | Optional: log, create internal record |
| `notification.transferStatusChanged` | Transfer status changes | `fromStatus`, `toStatus`, `settlementId` (when settlement is involved) | Update status; originator broadcasts only on `toStatus: "AUTHORIZED"` |
| `notification.transferAgentAdded` | An agent (intermediary, wallet service) is added | `agentID`, `agent { id, for, role }` | Update agents; a wallet service provisions and determines its role |
| `notification.transferAgentReplaced` | An agent is replaced | `fromAgentID`, `toAgentID`, `toAgent { id, for, role }` | Update agents |
| `notification.transferAgentStatusChanged` | One agent's status changes | `agentID`, `fromStatus`, `toStatus` | Track only; **not** a signal to settle |

Status values: [guide-travel-rule-api.md#transfer-statuses](guide-travel-rule-api.md#transfer-statuses). Agent roles in
`agent.role` / `toAgent.role`: `VASP`, `Custodian`, `SettlementAddress`, `SourceAddress`, `Gateway`.

Agent objects in these payloads use `id` (not `@id`):

```json
{
  "message": "notification.transferAgentReplaced",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "fromAgentID": "old-agent-123",
    "toAgentID": "new-agent-456",
    "toAgent": { "id": "new-agent-456", "for": "did:web:new-intermediary.com", "role": "VASP" }
  },
  "version": "1.0.0"
}
```

`transferAgentAdded` has the same shape with `agentID` and `agent` instead of `fromAgentID`/`toAgentID`/`toAgent`.

---

## TAP Policy Requirement Events

Notabene needs an action from you before the transfer can continue.

| `message` | Fires when | Key payload fields | What to do |
|---|---|---|---|
| `tap.requirePresentationRequested` | Counterparty's jurisdiction requires Travel Rule data | `originatorAgentID`, `beneficiaryAgentID`?, `originatorPresentationDefinition`, `beneficiaryPresentationDefinition`?, `callbackUrl`, `encryptionKey`?, `policyId` | Send IVMS101 PII via `callbackUrl` / [append](api-reference.md#pii) |
| `tap.requireAuthorizationRequested` | Policy, risk or business rules require an approve/reject decision | `originatorAgentID`, `beneficiaryAgentID`, `authorizeCallbackUrl`, `rejectCallbackUrl`, `purpose`? | POST to one callback: authorize with `{}`; reject with `{ "reason", "comment" }` ([reasons](guide-travel-rule-api.md#3-reject-a-transfer); `comment` required for `OTHER`) |
| `tap.requireRelationshipConfirmationRequested` | An address in the transfer must be confirmed as belonging to you / your client | `from` (counterparty agent DID), `originatorAgentID`, `beneficiaryAgentID`, `confirmCallbackUrl`, `purpose`? | Confirm via `confirmCallbackUrl` (the relationships confirm endpoint); don't confirm addresses that aren't yours |

`?` = optional. `beneficiary*` fields on `requirePresentationRequested` are present only when beneficiary data is
required.

```json
{
  "message": "tap.requirePresentationRequested",
  "payload": {
    "for": "did:web:your-vasp.com",
    "id": "transfer-123",
    "originatorAgentID": "did:web:your-vasp.com",
    "beneficiaryAgentID": "did:web:counterparty-vasp.com",
    "originatorPresentationDefinition": "https://pd.notabene.id/ivms101/v2/US-3000.json",
    "beneficiaryPresentationDefinition": "https://pd.notabene.id/ivms101/v2/JP-0.json",
    "callbackUrl": "/entities/did%3Aweb%3Aexample.com/transfers/transfer-123/presentations/callback",
    "encryptionKey": "...",
    "policyId": "policy-789"
  },
  "version": "1.0.0"
}
```

```json
{
  "message": "tap.requireAuthorizationRequested",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "originatorAgentID": "originator-agent-123",
    "beneficiaryAgentID": "beneficiary-agent-456",
    "authorizeCallbackUrl": "/entities/did%3Aweb%3Aexample.com/transfers/transfer-123/authorize",
    "rejectCallbackUrl": "/entities/did%3Aweb%3Aexample.com/transfers/transfer-123/reject",
    "purpose": "High-value transfer requires manual review"
  },
  "version": "1.0.0"
}
```

```json
{
  "message": "tap.requireRelationshipConfirmationRequested",
  "payload": {
    "for": "did:web:example.com",
    "from": "did:web:counterparty.com",
    "id": "transfer-123",
    "originatorAgentID": "originator-agent-123",
    "beneficiaryAgentID": "beneficiary-agent-456",
    "confirmCallbackUrl": "/entities/did%3Aweb%3Aexample.com/relationships?from=did%3Aweb%3Acounterparty.com&to=confirmed",
    "purpose": "Confirm relationship between entities"
  },
  "version": "1.0.0"
}
```

---

## TAP Policy Satisfaction Events

A previously requested action is complete.

| `message` | Fires when | Key payload fields | What to do |
|---|---|---|---|
| `tap.requireAuthorizationSatisfied` | An authorization decision was processed | `agentID` | Optional: audit log |
| `tap.requirePresentationSatisfied` | Required PII was presented and validated | `agentID`, `confirmBeneficiaryCallbackUrl`, `confirmInternalChecksCallbackUrl` | Beneficiary: verify name, then POST both callbacks (both required before authorization) |
| `tap.requireRelationshipConfirmationSatisfied` | Address relationship confirmed | `relationshipId`, `agentID`, `from`, `to`, `status` | Optional: update relationship records |

```json
{
  "message": "tap.requirePresentationSatisfied",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "agentID": "agent-456",
    "confirmBeneficiaryCallbackUrl": "/entities/did%3Aweb%3Aexample.com/tx/transfer-123/checks/beneficiary-name-matching",
    "confirmInternalChecksCallbackUrl": "/entities/did%3Aweb%3Aexample.com/tx/transfer-123/checks/internal-checks"
  },
  "version": "1.0.0"
}
```

---

## TAP Policy Status Events

| `message` | Fires when | Key payload fields | What to do |
|---|---|---|---|
| `tap.policySatisfied` | All requirements of a policy are met | `policyType` (`RequirePresentation`, `RequireAuthorization`, `RequireRelationshipConfirmation`) | Optional: compliance tracking |
| `tap.requirePresentationPartiallySatisfied` | Some but not all requested credentials received | `policyId` (UUID) | Optional: track progress; `tap.requirePresentationSatisfied` follows when complete |
