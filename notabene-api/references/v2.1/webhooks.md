# Webhooks

Notabene tells your systems when a transfer needs you, or has moved on without you. Events are delivered by
[Svix](https://docs.svix.com): signed, retried and replayable. Your endpoint is a plain HTTPS `POST` receiver.

## Quick start

Notabene calls your server when something needs you. Check the call is genuine, save it, then look at the transfer.

1. Add your endpoint under **Integration → Webhooks** and store its signing secret.
2. Verify each delivery with a Svix library, on the raw body.
3. Save the event, answer `2xx`, then re-read the transfer it names and act.

## 1. Register an endpoint

In the dashboard, open **Integration → Webhooks**, add your endpoint URL and choose the event types it receives. Copy
its signing secret (`whsec_…`). One endpoint per institution; send a test event from the same page.

## 2. Verify every request

Use an official Svix library with the **raw** request body and the `svix-id`, `svix-timestamp` and `svix-signature`
headers. Verify before parsing: re-serialised JSON fails verification.

```js
import { Webhook } from "svix";

const verifier = new Webhook(process.env.NOTABENE_WEBHOOK_SECRET);

app.post("/notabene/webhooks", express.raw({ type: "application/json" }), async (req, res) => {
  let event;
  try {
    event = verifier.verify(req.body, req.headers);
  } catch {
    return res.sendStatus(400); // bad signature: never retried into success
  }
  try {
    // Persist before acknowledging; the job id de-duplicates retries of the same event.
    await queue.add(event, { jobId: req.headers["svix-id"] });
  } catch {
    return res.sendStatus(500); // not saved: let Svix retry
  }
  res.sendStatus(204);
});
```

During secret rotation the old secret stays valid for a grace period.

## 3. Handle events safely

- **The body** is `{"eventType": "…", "payload": {…}}`. Every payload has `for` (your DID) and `id` (the transfer).
- **At-least-once, any order.** De-duplicate on `svix-id`; it is stable across retries.
- **Treat an event as a prompt, not as state.** Re-read `GET /api/v1/transfers/{id}` before acting; the API is the
  source of truth.
- **Acknowledge in seconds, after the event is saved.** Write it somewhere durable, answer `2xx`, then do the work
  asynchronously. Anything else is retried; the dashboard's message log can resend and replay.

## Events

Match on the full event type, including its `.v3` suffix. Full payload schemas are in the webhook events reference.

| Event | Act? | Meaning |
|---|---|---|
| `requirements.pending.v3` | **Yes** | Requirements wait on you. Each item has `id`, `type`, `status`, `from` and `agentId`; act on those whose `agentId` is your DID. |
| `requirements.presentation_requested.v3` | **Yes** | A counterparty asks for Travel Rule data; `presentationDefinition` describes what. |
| `requirements.authorization_requested.v3` | **Yes** | Your authorization is needed. |
| `relationship.confirmation_requested.v3` | **Yes** | Confirm a customer owns an address (`partyId`): resolve `wallet-address-confirmation` on the transfer. See [Wallet addresses](wallets.md). |
| `transfer.created.v3` | Usually | A transfer exists; `direction` is `Incoming` or `Outgoing` from your side. |
| `transfer.status_changed.v3` | Usually | `fromStatus` → `toStatus`. Outgoing + `ReadyToSettle` = broadcast now. `is_flagged` = held for review. |
| `requirements.presentation_satisfied.v3`, `requirements.authorization_satisfied.v3` | No | An outstanding requirement cleared. |
| `relationship.confirmed.v3` | No | Address ownership was confirmed. |
| `transfer.agent_added.v3`, `transfer.agent_replaced.v3`, `transfer.agent_removed.v3`, `transfer.agent_status_changed.v3` | No | The institutions on a transfer changed. |

Field casing is not uniform: `transfer.created.v3` sends `settlement_id`, `transfer.status_changed.v3` sends
`settlementId`. Copy names from the reference.

## Transfer statuses

| Status | Meaning |
|---|---|
| `Active` | Someone has not decided yet. |
| `ReadyToSettle` | Everyone authorized. The sender broadcasts and settles. |
| `Rejected`, `Canceled` | Someone refused. Do not broadcast. `Rejected` can still be lifted by re-evaluation. |
| `Settled` | Funds moved and everyone authorized. Done. |
| `SettledIncompleteAuthorization` | Funds moved; decisions still outstanding. Keep answering requirements. |
| `SettledCanceled` | Funds moved after a cancellation. Investigate. |
| `RevertRequested` … `Reverted` | A linked return transfer is under way or done. |

## An event did not arrive

1. Check the endpoint's message log under **Integration → Webhooks**: delivered, failed, or never sent.
2. Check the endpoint is enabled and subscribed to that event type.
3. Read the transfer. `requires_action: true` means something waits on you, whatever events you received.
