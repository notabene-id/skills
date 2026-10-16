---
name: notabene-api
description: >
  Integrate with Notabene Transact, Notabene's Travel Rule API. Use when writing, migrating or
  debugging code that calls Notabene endpoints or handles Notabene webhooks, when exchanging
  Travel Rule or IVMS101 data through Notabene, or when asking how a Notabene integration works.
  Covers API v1, v2 and v2.1, and the Notabene JavaScript SDK.
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
| Institution id | `vaspDID` (often `did:ethr:…`) | `entityDID` in the path | The token identifies it (a few routes such as `delegate-token` still take `entityDID`, under `/api/v1`) |
| Statuses | `NEW`, `SENT`, `ACK`, `ACCEPTED`, … | `OUTGOING`/`INCOMING`, `AUTHORIZED`, `SETTLED`, … | `Active`, `ReadyToSettle`, `Settled`, … |
| Webhooks | `notification.transactionUpdated` | `notification.*`, `tap.*` | `transfer.*.v3`, `requirements.*.v3`, `relationship.*.v3` |
| Amounts | Integer base units, asset symbols | Decimal strings, CAIP-19 | Whole-unit decimal strings, CAIP-19 |
| Broadcast after | `ACCEPTED` | `AUTHORIZED` | `ReadyToSettle` |
| Read | [`references/v1/README.md`](references/v1/README.md) | [`references/v2/README.md`](references/v2/README.md) | [`references/v2.1/README.md`](references/v2.1/README.md) |

v2.1's paths start with `/api/v1`. That is the v2.1 platform, not the v1 API: any `/api/v1/…` path means v2.1.

Test hosts are less telling than paths: `api.notabene.dev` is v1's test host and may also serve v2. When a host is
ambiguous, decide by path shape (`/tx/create` → v1, `/entities/{entityDID}/…` → v2).

New integrations go on v2.1 unless Notabene has told the institution otherwise. Existing integrations stay on their
version. To migrate, read both versions' READMEs and map every call, status and webhook event explicitly.

## Rules for every version

- Never broadcast a withdrawal before that version's "broadcast after" status, then report the transaction hash back.
- Webhooks arrive at least once and out of order: verify the Svix signature, make handlers idempotent, and re-read
  the transfer from the API instead of trusting the payload.
- Keep the API token server-side; front ends use per-customer tokens. Ask the user for secrets; never invent, print
  or commit them.
- Build against the test environment before production.
