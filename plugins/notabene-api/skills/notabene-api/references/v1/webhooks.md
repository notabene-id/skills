# Webhooks (v1)

v1 pushes every transaction change to one webhook URL per institution, delivered and signed by Svix. Each delivery
carries the whole transaction under `payload.transaction`; branch on its `status`. Deliveries can repeat, arrive out of
order and omit PII, so handlers must be idempotent and re-read when they need persons. Pitfalls:
[Caveats](caveats.md).

## Contents

- [Quick start](#quick-start)
- [Configure](#configure)
- [Delivery and verification](#delivery-and-verification)
- [Payload](#payload)
- [Events](#events)
- [Handling rules](#handling-rules)

## Quick start

1. `POST /webhook` with `{ "vaspDID", "url" }` (a fully qualified, dotted hostname); store the signing secret it
   returns.
2. Verify `svix-id`, `svix-timestamp`, `svix-signature` with a Svix library and that signing secret.
3. On `notification.transactionUpdated`, upsert by `id` (originators may use their `transactionRef`), act on
   `payload.transaction.status` ([Send](send.md), [Receive](receive.md)), and return 2xx.
4. Originators: broadcast on `notification.processBlockchainTransaction`, never after
   `notification.haltBlockchainTransaction` ([Send § 4](send.md#4-wait-for-the-safe-to-send-signal)).

## Configure

```
POST   /webhook            { "vaspDID": "did:ethr:0xbbbb…", "url": "https://you.example/notabene" }
                           → the endpoint's Svix signing secret, as the response body
GET    /webhook?vaspDID=…  → { "vaspDID", "url" }   (404 when none is configured)
DELETE /webhook            { "vaspDID": "did:ethr:0xbbbb…" }
```

Single-label hostnames (`https://webhooks/…`) are rejected.

### Deprecated: transaction notification callback

`POST /integrations/config/txnotification` with `{ vaspDID, callbackUrl }` (read via
`GET /integrations/txnotification/{vaspDID}`) is **deprecated**; use `/webhook`. It is independent of the Svix
webhook: on each change Notabene POSTs the **raw transaction JSON** (no envelope) to `callbackUrl` with only a
`Content-Type: application/json` header. These deliveries are **unsigned**: treat the body as a hint and re-read the
transaction with `GET /tx/info`. Non-2xx responses are retried, up to 15 attempts with exponential backoff starting at
10 seconds.

## Delivery and verification

Svix delivers every webhook with `svix-id`, `svix-timestamp` and `svix-signature` headers. Verify the raw body with
the signing secret `POST /webhook` returned before trusting it. Return 2xx once processed; any non-2xx makes Svix
redeliver. Notabene keeps one Svix application per institution, each endpoint with its own signing secret.

```javascript
import { Webhook } from "svix";
const wh = new Webhook(process.env.NOTABENE_V1_WEBHOOK_SECRET);
app.post("/notabene", express.raw({ type: "application/json" }), (req, res) => {
  let evt;
  try { evt = wh.verify(req.body, req.headers); } catch { return res.status(400).end(); }
  // evt.message, evt.payload.transaction
  res.status(200).end();
});
```

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
  "version": "1.0.0"
}
```

Deliveries to the beneficiary side omit `transactionRef` (the originator's private field). By default Notabene strips
`pii.originator`, `pii.beneficiary`, `ivms101.originator` and `ivms101.beneficiary` from these payloads (a
Notabene-side flag controls it); otherwise persons may be nameless placeholders. Persons sit under `ivms101` (or
`pii`), never top-level `originator` / `beneficiary`.

Who is notified: the originator institution on every change. The beneficiary institution only when the status is
`INCOMPLETE`, `SENT`, `ACK`, `ACCEPTED`, `DECLINED`, `REJECTED` or `NOT_READY`, when a transaction it could already
see is `CANCELLED`, or when a `NON_CUSTODIAL` transaction is `SAVED`.

## Events

`message` is the event type; each carries exactly one transaction.

| `message` | When |
| --- | --- |
| `notification.transactionUpdated` | Every status or data change after creation. Drives the lifecycle. |
| `notification.processBlockchainTransaction` | Originator only, when the institution's `statusToProcessBlockchain` setting is set: the transfer reached the configured status (or `SAVED`, `NOT_READY`, creation; see [Send § 4](send.md#4-wait-for-the-safe-to-send-signal)). Safe to broadcast. |
| `notification.haltBlockchainTransaction` | Originator only: the transfer became `CANCELLED`, `DECLINED` or `REJECTED`. Do not broadcast. |

## Handling rules

- **PII may be stripped.** Re-read `GET /tx/info?id=…` when you need the persons.
- **At-least-once, unordered.** De-duplicate on `svix-id` and key handlers on `id` + `status`; never move your own
  state backwards when a late, older status arrives.
- **Webhooks can beat your call's response.** Originators correlate on the `transactionRef` set on create, so the
  record exists before the call returns. Beneficiaries never see it (redacted from their webhooks) and key on `id`.
- **`pending-` hashes are placeholders.** A `transactionBlockchainInfo.txHash` starting with `pending-` is seeded by
  some Notabene components before settlement; it is not an on-chain hash.
