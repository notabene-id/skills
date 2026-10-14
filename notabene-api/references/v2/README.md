# Notabene API v2

Route for integrations on `https://api.eu1.notabene.id` or `https://api.us1.notabene.id`, with paths under `/entities/{entityDID}/…`.

Read [`guide-travel-rule-api.md`](guide-travel-rule-api.md) end to end before writing code: it is the one home for
authentication, identifiers, webhook setup and verification, and statuses.

| Need | Guide |
|---|---|
| Outgoing and incoming transfers, PII, authorize/reject, settle | [`guide-travel-rule-api.md`](guide-travel-rule-api.md) |
| Every endpoint, request and response schema | [`api-reference.md`](api-reference.md) |
| Every webhook event and payload | [`webhooks.md`](webhooks.md) |
| Wallet service / custody provider acting as an agent for another institution | [`wallet-service-guide.md`](wallet-service-guide.md) |
| Embedding Travel Rule UI (withdrawal, deposit, wallet verification) | [`guide-javascript-sdk.md`](guide-javascript-sdk.md) |

**Originator lifecycle:**
1. `POST /entities/{entityDID}/tx` — create the transfer **before** broadcasting.
2. On `tap.requirePresentationRequested` → `POST .../transfers/{id}/append` with IVMS101 PII.
3. Wait for `notification.transferStatusChanged` with `toStatus: "AUTHORIZED"`. Never broadcast before it.
4. `POST .../tx/{id}/settle` with `{ "settlementId": "eip155:{chainId}:tx/{txHash}" }`.

**Beneficiary lifecycle:**
1. Register deposit addresses via `POST /entities/{entityDID}/relationships` (recommended; avoids a confirmation per
   transfer).
2. On `tap.requirePresentationSatisfied` → `GET` the transfer with `?decrypt=true`, run name matching.
3. Complete `/checks/beneficiary-name-matching` and `/checks/internal-checks`.
4. Authorize through configured policies or `POST .../tx/{id}/authorize` (empty body `{}`), or reject.
5. Deposits seen on-chain first: find the transfer with `GET .../tx/match`.

**Wallet services (agents acting for another institution):** follow the transfer's `agents` `{ agent, for }` chain
from your DID; reaching `originator` means you send, reaching `beneficiary` means you receive. Register client
addresses with the `custodian` field set to your own DID. Use an atomic claim so duplicate `AUTHORIZED` deliveries
never settle twice.

**JavaScript SDK** — `npm install @notabene/javascript-sdk`. Generate a `customerToken` server-side (never ship the API
token to the browser), render `createWithdrawalAssist` / `createDepositAssist` / `createDepositRequest` /
`createConnectWallet`, and use the response's `txCreate` as the body of `POST /entities/{entityDID}/tx`.
