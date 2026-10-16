# Notabene API v1 (legacy)

Route for integrations on `https://api.notabene.id` (test: `https://api.notabene.dev`): transactions keyed by
`vaspDID`, created with `POST /tx/create` and driven through `/tx/*` verbs. Treat v1 code as maintenance: fix it in v1
terms, and propose migration to v2.1 rather than extending it with new features.

| Need | Guide |
|---|---|
| Recognising v1, auth, errors, the transaction object, statuses, which verb when | [`getting-started.md`](getting-started.md) |
| Where to hook Notabene into the codebase: the pre-broadcast withdrawal hook, registering addresses as wallets are created, backfilling existing addresses, the deposit path | [`integration-points.md`](integration-points.md) |
| Withdrawals (originator) | [`send.md`](send.md) |
| Deposits (beneficiary), `/tx/notify`, address registration | [`receive.md`](receive.md) |
| Webhook setup, verification, payload, events | [`webhooks.md`](webhooks.md) |
| Gotchas, and the facts that could not be confirmed | [`caveats.md`](caveats.md) |

**Originator:** optional `POST /tx/validate/full` → `POST /tx/create` (set a unique `transactionRef`) → `/tx/approve`
unless an auto-approve rule does it (`NEW` → `SENT`) → wait for the beneficiary institution (`SENT` → `ACK` →
`ACCEPTED`) → broadcast on the Svix event `notification.processBlockchainTransaction` → record the hash with
`POST /tx/update { "id", "txHash" }`. That event fires when the transfer reaches the institution's
`statusToProcessBlockchain` setting (usually `ACCEPTED`); `notification.haltBlockchainTransaction` means do not send.
Without the setting there is no go-ahead event: broadcast only after `ACCEPTED`. A create that stays `SAVED` (below
threshold, self-hosted or internal) is never sent: broadcast and record the hash without waiting.

**Beneficiary:** register deposit addresses → on `SENT`, `/tx/confirm` the address (→ `ACK`) or `/tx/reject` →
`/tx/accept` (only from `ACK`; confirming is not accepting) or `/tx/decline` (from `ACK`, `SENT`, `SAVED`,
`INCOMPLETE`). A deposit seen on-chain first is reported with `POST /tx/notify`, whose hash is the settlement.

**Gotchas:** amounts are integer base units (`"1000000000000000000"` = 1 ETH), not decimals; a 200 from `/tx/update` can
be a silent no-op and `/tx/info` answers an empty 200 for a transaction you cannot see, so re-read and check; webhooks
may omit PII (re-read with `/tx/info`); bodies are camelCase except `vasp_did` on `/tx/cancel` and `/v1/addresses`;
IVMS uses `nameIdentifierType`, not v2.1's `naturalPersonNameIdentifierType`.
