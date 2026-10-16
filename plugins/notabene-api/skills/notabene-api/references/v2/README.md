# Notabene API v2

Route for integrations on `https://api.eu1.notabene.id` or `https://api.us1.notabene.id`, with paths under `/entities/{entityDID}/…`.

Read [`guide-travel-rule-api.md`](guide-travel-rule-api.md) end to end before writing code: it is the one home for
authentication, identifiers, webhook setup and verification, and statuses.

| Need | Guide |
|---|---|
| Outgoing and incoming transfers, PII, authorize/reject, settle | [`guide-travel-rule-api.md`](guide-travel-rule-api.md) |
| Endpoints for the flows above, request and response schemas | [`api-reference.md`](api-reference.md) |
| Every webhook event and payload | [`webhooks.md`](webhooks.md) |
| Wallet service / custody provider acting as an agent for another institution | [`wallet-service-guide.md`](wallet-service-guide.md) |
| Embedding Travel Rule UI (withdrawal, deposit, wallet verification) | [`guide-javascript-sdk.md`](guide-javascript-sdk.md) |

**Originator lifecycle:**
1. `POST /entities/{entityDID}/tx` — create the transfer **before** broadcasting.
2. On `tap.requirePresentationRequested` → POST the IVMS101 PII to the payload's `callbackUrl` (`.../tx/{id}/append`,
   or `.../policies/{policyId}/presentation` for entity-managed encryption).
3. Wait for `notification.transferStatusChanged` with `toStatus: "AUTHORIZED"`. Never broadcast before it.
4. `POST .../tx/{id}/settle` with `{ "settlementId": "eip155:{chainId}:tx/{txHash}" }`.

Nothing is delivered until webhooks are enabled with `PUT /entities/{entityDID}/webhook` and an endpoint is added in
the Svix portal ([setup](guide-travel-rule-api.md#webhook-setup)).

**Beneficiary lifecycle:**
1. Register deposit addresses via `POST /entities/{entityDID}/relationships` (recommended). A relationship naming your
   own DID as `from`, `to` or `custodian` is confirmed on creation; otherwise confirm with `PATCH` on the
   `confirmCallbackUrl` of `tap.requireRelationshipConfirmationRequested`.
2. On `tap.requirePresentationSatisfied` → `GET` the transfer with `?decrypt=true`, run name matching.
3. Complete the checks your entity has configured (`/checks/beneficiary-name-matching`, `/checks/internal-checks`);
   their callback URLs appear in the payload only when configured.
4. Authorize through configured policies or `POST .../tx/{id}/authorize` (`{}`, or
   `{ "settlementAddress": "<CAIP-10>" }` with optional `memoTag`), or reject.
5. Deposits seen on-chain first: find the transfer with `GET .../tx/match`.

**Wallet services (agents acting for another institution):** follow the transfer's `agents` `{ agent, for }` chain
from your DID; reaching `originator` means you send, reaching `beneficiary` means you receive. Register client
addresses with the `custodian` field set to your own DID. Use an atomic claim so duplicate `AUTHORIZED` deliveries
never settle twice.

**JavaScript SDK** — `npm install @notabene/javascript-sdk`. Get a customer token server-side from
`POST /entities/{entityDID}/delegateToken` with `{ entityDID, delegateId }` (valid 1 hour; never ship the API token to
the browser), render `createWithdrawalAssist` / `createDepositAssist` / `createDepositRequest` / `createConnectWallet`,
then convert the result with `componentResponseToTxRequests(response, token, config)`: POST `createTx` to
`/entities/{entityDID}/tx` and `ivms101` to `.../tx/{id}/append`. The response's `txCreate` is v1-shaped; never send
it to v2.
