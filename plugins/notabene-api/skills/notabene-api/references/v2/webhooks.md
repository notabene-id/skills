# Notabene Webhook Events (v2)

Catalogue of every webhook event. Mechanics live in the main guide:

- Setup (`PUT /entities/{entityDID}/webhook`, then add endpoints in the Svix portal; nothing is sent before that):
  [guide-travel-rule-api.md#webhook-setup](guide-travel-rule-api.md#webhook-setup)
- Svix signature verification:
  [guide-travel-rule-api.md#signature-verification](guide-travel-rule-api.md#signature-verification)
- Envelope `{ message, payload, version }` (event type in `message`, transfer id in `payload.id`):
  [guide-travel-rule-api.md#webhook-envelope](guide-travel-rule-api.md#webhook-envelope)
- At-least-once, out-of-order delivery and idempotent handlers:
  [guide-travel-rule-api.md#delivery-guarantees](guide-travel-rule-api.md#delivery-guarantees)

Every `payload` carries `for` (DID of the entity the webhook is for) and `id` (transfer id); the tables list the other
fields. Callback URLs are relative paths, mostly with unencoded DIDs: prepend the API base URL and URL-encode any DID
not already encoded (some still use the deprecated `/transfers/...` prefix, an alias of `/tx/...`). Respond `2xx` once
processed; any other status or a timeout is retried.

## Contents

- [Transfer Management Events](#transfer-management-events)
- [TAP Policy Requirement Events](#tap-policy-requirement-events)
- [TAP Policy Satisfaction Events](#tap-policy-satisfaction-events)
- [TAP Policy Status Events](#tap-policy-status-events)
- [Settlement Signal Event](#settlement-signal-event)

---

## Transfer Management Events

| `message` | Fires when | Key payload fields | What to do |
|---|---|---|---|
| `notification.transferCreated` | A transfer is created and enters processing | — | Optional: log, create internal record |
| `notification.transferStatusChanged` | Transfer status changes | `fromStatus`, `toStatus`, `settlementId` (when settlement is involved) | Update status; originator broadcasts only on `toStatus: "AUTHORIZED"` |
| `notification.transferAgentAdded` | An agent (intermediary, wallet service) is added | `agentID`, `agent { id, for, role }` | Update agents; a wallet service provisions and determines its role |
| `notification.transferAgentReplaced` | An agent is replaced | `fromAgentID`, `toAgentID`, `toAgent { id, for, role }` | Update agents |
| `notification.transferAgentRemoved` | An agent is removed ([`agents/remove`](api-reference.md#transfers)) | `removedAgentID`, `reason`? | Update agents |
| `notification.transferAgentStatusChanged` | One agent's status changes (e.g. `PROCESSING` → `AUTHORIZED`) | `agentID`, `fromStatus`, `toStatus` | Track only; **not** a signal to settle |

Transfer status values (`AWAITING-YOURS` for new transfers, `AWAITING-COUNTERPARTY`, `AUTHORIZED`, …; no `PENDING`):
[guide-travel-rule-api.md#transfer-statuses](guide-travel-rule-api.md#transfer-statuses). Agent statuses:
`PROCESSING`, `AUTHORIZED`, `REJECTED`, `SETTLED`, `FAILED`, `REPLACED`, `UNREACHABLE`. Agent roles in
`agent.role` / `toAgent.role`: `VASP`, `Custodian`, `SettlementAddress`, `SourceAddress`, `Gateway`, `Unknown`.

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
| `tap.requirePresentationRequested` | Counterparty's jurisdiction requires Travel Rule data | `originatorAgentID`, `beneficiaryAgentID`?, `originatorPresentationDefinition` (may be empty), `beneficiaryPresentationDefinition`?, `callbackUrl`, `encryptionKey`?, `policyId` | POST the IVMS101 PII to `callbackUrl`: `…/transfers/{id}/append` (Notabene-managed encryption) or `…/transfers/{id}/policies/{policyId}/presentation` (entity-managed) ([PII endpoints](api-reference.md#pii)) |
| `tap.requireAuthorizationRequested` | Policy, risk or business rules require an approve/reject decision | `originatorAgentID`, `beneficiaryAgentID`, `authorizeCallbackUrl`, `rejectCallbackUrl`, `purpose`? | POST to one callback with the authorize / reject endpoint's body: authorize with `{}`; reject with `{ "reason", "comment" }` (`reason` required; [reasons](guide-travel-rule-api.md#3-reject-a-transfer); `comment` required for `OTHER`) |
| `tap.requireRelationshipConfirmationRequested` | An address in the transfer must be confirmed as belonging to you / your client | `from` (the agent, usually the address DID, whose relationship must be confirmed), `originatorAgentID`, `beneficiaryAgentID`, `confirmCallbackUrl` (with `from`/`to` query), `purpose`? | `PATCH` the `confirmCallbackUrl` (`PATCH /entities/{entityDID}/relationships?from=…&to=…`); don't confirm addresses that aren't yours |

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
    "callbackUrl": "/entities/did:web:example.com/transfers/transfer-123/append",
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
    "authorizeCallbackUrl": "/entities/did:web:example.com/transfers/transfer-123/authorize",
    "rejectCallbackUrl": "/entities/did:web:example.com/transfers/transfer-123/reject",
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

---

## TAP Policy Satisfaction Events

A previously requested action is complete.

| `message` | Fires when | Key payload fields | What to do |
|---|---|---|---|
| `tap.requireAuthorizationSatisfied` | One agent's authorization requirement is satisfied (per agent, not the whole transfer) | `agentID` | Optional: audit log |
| `tap.requirePresentationSatisfied` | Required PII was presented and validated; sent to both parties | `agentID`, `policyType` (`REQUIRE_PRESENTATION`, or `COUNTERPARTY_REQUIRE_PRESENTATION` on the counterparty's copy), `confirmBeneficiaryCallbackUrl`?, `confirmInternalChecksCallbackUrl`? | Beneficiary: verify name, then POST each callback present (a callback appears only when that check is configured; pending checks block authorization) |
| `tap.requireRelationshipConfirmationSatisfied` | Address relationship confirmed | `relationshipId`, `agentID` (same as `from`), `from` (address DID), `to` (entity DID), `status` (`CONFIRMED` or `PROVEN`) | Optional: update relationship records |

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

---

## TAP Policy Status Events

| `message` | Fires when | Key payload fields | What to do |
|---|---|---|---|
| `tap.requirePresentationPartiallySatisfied` | Some but not all requested credentials received | `policyId` (UUID) | Optional: track progress; `tap.requirePresentationSatisfied` follows when complete |

There is no `tap.policySatisfied` event.

---

## Settlement Signal Event

| `message` | Fires when | Key payload fields | What to do |
|---|---|---|---|
| `notification.settleTransaction` | Only for entities with a settlement policy for which Notabene has enabled settlement signals: a transfer is ready to settle or needs action | `transferId`, `trigger` (`INTERNAL_APPROVED`, `AUTHORIZATION_COMPLETE`, `COUNTERPARTY_REJECTED`, `TIMEOUT`), `action` (`SETTLE`, `REVIEW`, `CANCEL`, `ACK`), `transferStatus`, `settlementReadiness`, `internalPoliciesComplete`, `authorizationRequirementsComplete`, `authorizationRequirementsPending[]` (`requirement`, `from`, `responded`, `timedOut`, `timeoutAt`?), `timedOut`, `timeoutAction`? (`AUTHORIZE`, `FLAG`, `REJECT`), `counterpartyRejectedAction`? (`FLAG`, `REJECT`) | Act on `action` |
