# Notabene API v2.1 (current platform)

Route for integrations on `https://sandbox.notabene.id` (sandbox) or `https://eu-a.notabene.id` /
`https://us-a.notabene.id` (production, EU and US), with paths under `/api/v1/…`.

Read [`authentication.md`](authentication.md) first (environments, API credential, token, conventions, errors), then
[`overview.md`](overview.md) (the model in payment terms, objects, lifecycle), then the guides the path needs. The API
reference, served at `/api/openapi.json` on sandbox (`https://sandbox.notabene.id/api/openapi.json`) and in the
dashboard under **Integration → API Reference**, is authoritative for field names, enums and permissions.

**1. Find the integration type.** Settle it before writing code (infer from the codebase; ask only what you cannot
see): direction (withdrawals, deposits or both); who holds the wallets (a wallet provider connected in the dashboard →
wallet-managed, otherwise self-managed); whether the front end runs live checks (→ embedded).
[`go-live.md`](go-live.md#what-each-answer-means) ("What each answer means") holds the decision table and the minimal
call/event set per direction. If the user has already saved an integration type in the dashboard (**Integration → Go
Live**), reconcile with it: build for the saved type, or tell the user why it should change.

**2. Build the path**, against sandbox, in the order [`go-live.md`](go-live.md) lists the steps:

| Path | Read |
|---|---|
| Every path | [`authentication.md`](authentication.md), then [`webhooks.md`](webhooks.md) — endpoint, Svix verification, events, statuses |
| Customers (every path) | [`customers.md`](customers.md) — register once, ids on transfers, IVMS-101 minimums |
| Deposit addresses (every path that receives) | [`wallets.md`](wallets.md) — CAIP-10 registration, confirmation requests, self-hosted wallet proofs |
| Withdrawals | [`transact-send.md`](transact-send.md) — assess, create, requirements, `ReadyToSettle`, settle |
| Deposits | [`transact-receive.md`](transact-receive.md) — announced before settlement, or seen on-chain first |
| Embedded front end | [`transact-embedded.md`](transact-embedded.md) — customer tokens and the customer routes |

Done when a sandbox transfer reaches `Settled`.

**3. Production.** Walk [`pre-production.md`](pre-production.md) against the code and report each check as done,
missing or not applicable. [`go-live.md`](go-live.md) is the setup checklist; the dashboard's **Integration → Go Live**
tracks the same steps per environment and stores the saved integration type, so every step is done again in
production.

**Invariants worth checking in any v2.1 code:**

- Every transfer create carries a persisted UUID `ref`; retries replay the identical body.
- Every transfer names each customer by id (`originator_id`, `beneficiary_id`); registered customers are named by id
  alone (recommended). An inline profile (`originator`, `beneficiary`) goes alongside its id, never instead of it: an
  inline profile without its id is rejected, and `ivms_encrypted` always needs its id. Customer creation is keyed by
  `id` and answers `409` when the customer already exists.
- IVMS-101 keys are IVMS's own camelCase (`naturalPersonNameIdentifierType`, not `nameIdentifierType`).
- `POST /api/v1/wallets` takes CAIP-10 only (`eip155:1:0x…`); transfers also accept bare addresses.
- Amounts are whole-unit decimal strings (`"1.5"`, never wei, never a float).
- Withdrawals broadcast only after `ReadyToSettle`, then call settle with the transaction hash.
- Webhooks are verified on the raw body with Svix, de-duplicated on `svix-id`, and re-read state from the API. Event
  types carry the `.v3` suffix: `transfer.*.v3` (`created`, `status_changed`, `agent_added`, `agent_replaced`,
  `agent_removed`, `agent_status_changed`), `requirements.*.v3` (`pending`, `presentation_requested`,
  `presentation_satisfied`, `authorization_requested`, `authorization_satisfied`) and `relationship.*.v3`
  (`confirmation_requested`, `confirmed`).
- `SettledIncompleteAuthorization` is unfinished; `requires_action: true` (on `GET /api/v1/transfers/{id}`) means
  something waits on the integrator.
- Setup happens in the dashboard, not in code: API credential (Integration → API Credentials, secret shown once),
  webhook endpoint and its `whsec_` secret (Integration → Webhooks), wallet provider (Integration → Wallet Providers).
  Ask the user for secrets; never invent, print or commit them.
