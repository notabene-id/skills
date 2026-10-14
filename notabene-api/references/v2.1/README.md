# Notabene API v2.1 (current platform)

Route for integrations on `https://sandbox.notabene.id`, `https://eu-a.notabene.id` or `https://us-a.notabene.id`,
with paths under `/api/v1/…`.

Read [`getting-started.md`](getting-started.md) first, then the guides the path needs.
The API reference served at `/api/openapi.json` on sandbox (and in the dashboard under Integration → Getting Started →
API Reference) is authoritative for field names, enums and permissions.

**1. Find the integration.** Settle before writing code (infer from the codebase; ask only what you cannot see):
direction (withdrawals, deposits or both); who holds the wallets (a wallet provider connected in the dashboard →
wallet-managed, otherwise self-managed); whether the front end runs live checks (→ embedded).
[`choose-your-integration.md`](choose-your-integration.md) holds the decision tables and the minimal
call/event set per path.

**2. Build the path**, against sandbox, in the guide's order:

| Path | Read |
|---|---|
| Every path | [`getting-started.md`](getting-started.md) — environments, credential, token, webhook endpoint, conventions, errors |
| The model, for someone new to it | [`overview.md`](overview.md) — the Travel Rule in payment terms, objects, lifecycle |
| Customers (every path) | [`customers.md`](customers.md) — register once, ids on transfers, IVMS-101 minimums |
| Deposit addresses (every path that receives) | [`wallets.md`](wallets.md) — CAIP-10 registration, confirmation requests, self-hosted wallet proofs |
| Withdrawals | [`transact-send.md`](transact-send.md) |
| Deposits | [`transact-receive.md`](transact-receive.md) |
| Embedded front end | [`transact-embedded.md`](transact-embedded.md) |
| Event handling | [`webhooks.md`](webhooks.md) |

Done when a sandbox transfer reaches `Settled`.

**3. Production.** Walk [`go-live.md`](go-live.md) against the code and report each unchecked item.

**Invariants worth checking in any v2.1 code:**

- Every transfer create carries a persisted UUID `ref`; retries replay the identical body.
- Every transfer names both customers by id (`originator_id`, `beneficiary_id`); an inline IVMS profile without its
  id is rejected. Customer creation is keyed by `id` and answers `409` when the customer already exists.
- IVMS-101 keys are IVMS's own camelCase (`naturalPersonNameIdentifierType`, not `nameIdentifierType`).
- `POST /api/v1/wallets` takes CAIP-10 only (`eip155:1:0x…`); transfers also accept bare addresses.
- Amounts are whole-unit decimal strings (`"1.5"`, never wei, never a float).
- Withdrawals broadcast only after `ReadyToSettle`, then call settle with the transaction hash.
- Webhooks are verified on the raw body with Svix, de-duplicated on `svix-id`, and re-read state from the API.
- `SettledIncompleteAuthorization` is unfinished; `requires_action: true` means something waits on the integrator.
- Setup happens in the dashboard, not in code: API credential (Integration → API Credentials, secret shown once),
  webhook endpoint and its `whsec_` secret (Integration → Webhooks), wallet provider (Integration → Wallet Providers).
  Ask the user for secrets; never invent, print or commit them.
