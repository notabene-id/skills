---
name: notabene-api
description: >
  Integrate with Notabene Transact, Notabene's Travel Rule API. Use when writing, migrating or debugging code
  that calls Notabene endpoints or handles Notabene webhooks, when adding Travel Rule checks to withdrawals,
  deposits or wallet creation, when exchanging Travel Rule or IVMS101 data through Notabene, or when asking how a
  Notabene integration works. Covers API v1, v2 and v2.1, and the Notabene JavaScript SDK.
---

# Notabene API

Notabene Transact has three API generations. They differ in base URL, paths, payloads, statuses and webhooks. Work
out which one the integrator is on, then read only that version's `README.md`. Never mix calls or field names
across versions. Flow (payments) is not covered.

## Which version?

Infer it from the codebase, config or credentials before asking. Ask only when nothing shows it.

| Signal | v1 (legacy) | v2 | v2.1 (current platform) |
|---|---|---|---|
| Base URL | `api.notabene.id` | `api.eu1.notabene.id`, `api.us1.notabene.id` | `sandbox.notabene.id`, `eu-a.notabene.id`, `us-a.notabene.id` |
| Paths | `/tx/create`, `/tx/info`, `/tx/update` | `/entities/{entityDID}/tx` | `/api/v1/transfers`, `/api/v1/customers` |
| Institution id | `vaspDID`, usually `did:ethr:…` | `entityDID` in the path, usually `did:web:…` | Usually `did:web:…`; the token identifies it (a few routes such as `delegate-token` still take `entityDID`, under `/api/v1`) |
| Statuses | `NEW`, `SENT`, `ACK`, `ACCEPTED`, … | `AWAITING-YOURS`, `AWAITING-COUNTERPARTY`, `AUTHORIZED`, `SETTLED`, … | `Active`, `ReadyToSettle`, `Settled`, … |
| Webhooks | `notification.transactionUpdated`, `notification.processBlockchainTransaction` | `notification.*`, `tap.*` | `transfer.*.v3`, `requirements.*.v3`, `relationship.*.v3` |
| Amounts | Integer base units, asset symbols | Decimal strings, CAIP-19 | Whole-unit decimal strings, CAIP-19 |
| Broadcast after | `notification.processBlockchainTransaction`, fired at the institution's `statusToProcessBlockchain` setting (usually `ACCEPTED`). Without it: `ACCEPTED`, or `SAVED` (not sent: below threshold, self-hosted or internal) | `AUTHORIZED` | `ReadyToSettle` |
| Read | [`references/v1/README.md`](references/v1/README.md) | [`references/v2/README.md`](references/v2/README.md) | [`references/v2.1/README.md`](references/v2.1/README.md) |

v2.1's paths start with `/api/v1`. That is the v2.1 platform, not the v1 API: any `/api/v1/…` path means v2.1.

`api.notabene.dev` is v1's test host and accepts only v1 tokens. v2 has no separate test host: a v2 sandbox entity
uses `api.eu1.notabene.id`.

New integrations go on v2.1 unless Notabene has told the institution otherwise. Existing integrations stay on their
version. To migrate, read both versions' READMEs and map every call, status and webhook event explicitly.

## Where it goes in the codebase

Whatever the version, the integration lives in four places in the integrator's code. Find them before writing any
call, then follow that version's `integration-points.md`:

1. **Withdrawals:** a pre-authorization hook after the withdrawal is validated and before it is signed or broadcast.
   The withdrawal waits in its own state until the "broadcast after" signal, and is cancelled on a rejection.
2. **New wallets:** register every deposit address with its customer when it is created or allocated.
3. **Existing addresses:** a one-off, resumable backfill of the addresses already issued.
4. **Deposits:** when funds arrive, match them to the incoming transfer, then credit or hold per the version's guide
   and the institution's policy.

## Rules for every version

- Never broadcast a withdrawal before that version's "broadcast after" signal (v1: never after
  `notification.haltBlockchainTransaction`), then report the transaction hash back.
- IVMS101 keys differ: a natural person's name type is `nameIdentifierType` in v1 and
  `naturalPersonNameIdentifierType` in v2 and v2.1.
- Webhooks arrive at least once and out of order: verify the Svix signature, make handlers idempotent, and re-read
  the transfer from the API instead of trusting the payload.
- Keep the API token server-side; front ends use per-customer tokens. Ask the user for secrets; never invent, print
  or commit them.
- Build against the test environment before production.
