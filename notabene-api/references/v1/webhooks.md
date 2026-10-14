# Webhooks (v1)

v1 pushes every transaction change to one webhook URL per institution, delivered and signed by Svix. Each delivery
carries the whole transaction under `payload.transaction`; branch on its `status`. Deliveries can repeat, arrive out of
order and omit PII, so handlers must be idempotent and re-read when they need persons. See [Caveats](caveats.md) for the
open questions (signing secret, `version`).

## Quick start

1. `POST /webhook` with `{ "vaspDID", "url" }` (a fully qualified, dotted hostname).
2. Verify `svix-id`, `svix-timestamp`, `svix-signature` with a Svix library and your endpoint's signing secret.
3. On `notification.transactionUpdated`, upsert by `transactionRef` / `id`, act on `payload.transaction.status`
   ([Send](send.md), [Receive](receive.md)), and return 2xx.

## Configure

```
POST   /webhook            { "vaspDID": "did:ethr:0xbbbb…", "url": "https://you.example/notabene" }
GET    /webhook?vaspDID=…  → { "vaspDID", "url" }   (404 when none is configured)
DELETE /webhook            { "vaspDID": "did:ethr:0xbbbb…" }
```

Single-label hostnames (`https://webhooks/…`) are rejected. `POST /integrations/config/txnotification` with
`{ vaspDID, callbackUrl }` (read via `GET /integrations/txnotification/{vaspDID}`) is **deprecated**; use `/webhook`.

## Delivery and verification

Svix delivers every webhook with `svix-id`, `svix-timestamp` and `svix-signature` headers. Verify before trusting the
body. Return 2xx once processed; any non-2xx makes Svix redeliver. Notabene keeps one Svix application per institution,
each endpoint with its own signing secret.

## Payload

```json
{
  "message": "notification.transactionUpdated",
  "payload": {
    "transaction": {
      "id": "6f1c…-uuid",
      "transactionRef": "withdrawal-8841",
      "status": "ACK",
      "transactionAsset": "ETH",
      "transactionAmount": "1000000000000000000",
      "originatorVASPdid": "did:ethr:0xaaaa…",
      "beneficiaryVASPdid": "did:ethr:0xbbbb…",
      "transactionBlockchainInfo": { "txHash": "0x9c1d…", "destination": "0xBeneficiaryAddress" }
    }
  },
  "version": "…"
}
```

## Events

`message` is the event type; each carries exactly one transaction.

| `message` | When |
| --- | --- |
| `notification.transactionUpdated` | Every status or data change after creation. Drives the lifecycle. |
| `notification.processBlockchainTransaction` | v1 saw the originating on-chain transfer and asks you to record or process it. |
| `notification.haltBlockchainTransaction` | A downstream signal (sanctions, risk score, manual hold) took the transfer out of the active path. Stop processing it. |

## Handling rules

- **PII may be stripped.** Re-read `GET /tx/info?id=…` when you need the persons.
- **At-least-once, unordered.** Key handlers on `id` + `status`; never move your own state backwards when a late,
  older status arrives.
- **Webhooks can beat your call's response.** Correlate on the `transactionRef` you set on create, so the record exists
  before the call returns.
- **`pending-` hashes are placeholders.** A `transactionBlockchainInfo.txHash` starting with `pending-` is seeded by
  some Notabene components before settlement; it is not an on-chain hash.
