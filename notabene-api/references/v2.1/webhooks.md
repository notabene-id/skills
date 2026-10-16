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
headers. Verify before parsing: re-serialised JSON fails verification. Each example answers `400` to a bad signature,
saves the event keyed by `svix-id`, and answers `500` if saving fails so Svix retries.

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

```go
import svix "github.com/svix/svix-webhooks/go"

verifier, err := svix.NewWebhook(os.Getenv("NOTABENE_WEBHOOK_SECRET"))
if err != nil {
	log.Fatal(err)
}

http.HandleFunc("POST /notabene/webhooks", func(w http.ResponseWriter, r *http.Request) {
	payload, err := io.ReadAll(r.Body)
	if err != nil || verifier.Verify(payload, r.Header) != nil {
		w.WriteHeader(http.StatusBadRequest) // bad signature: never retried into success
		return
	}
	// Persist before acknowledging; the svix-id de-duplicates retries of the same event.
	if err := queue.Add(r.Context(), r.Header.Get("svix-id"), payload); err != nil {
		w.WriteHeader(http.StatusInternalServerError) // not saved: let Svix retry
		return
	}
	w.WriteHeader(http.StatusNoContent)
})
```

```python
from svix.webhooks import Webhook, WebhookVerificationError

verifier = Webhook(os.environ["NOTABENE_WEBHOOK_SECRET"])

@app.post("/notabene/webhooks")
def notabene_webhook():
    try:
        event = verifier.verify(request.get_data(), request.headers)
    except WebhookVerificationError:
        return "", 400  # bad signature: never retried into success
    try:
        # Persist before acknowledging; the svix-id de-duplicates retries of the same event.
        queue.add(event, job_id=request.headers["svix-id"])
    except Exception:
        return "", 500  # not saved: let Svix retry
    return "", 204
```

During secret rotation the old secret stays valid for a grace period.

## 3. Handle events safely

- **The body** is `{"eventType": "…", "payload": {…}}`. Every payload has `for` (your DID) and `id` (the transfer).
- **At-least-once, any order.** De-duplicate on `svix-id`; it is stable across retries.
- **Treat an event as a prompt, not as state.** Re-read `GET /api/v1/transfers/{id}` before acting; the API is the
  source of truth.
- **Acknowledge in seconds, after the event is saved.** Write it somewhere durable, answer `2xx`, then do the work
  asynchronously. Anything else is retried; the dashboard's message log can resend and replay.
- **Sweep for missed work.** A scheduled job pages `GET /api/v1/transfers` (for example `status=Active` and
  `status=SettledIncompleteAuthorization`) and reads each with `GET /api/v1/transfers/{id}`: `requires_action` is on
  the single-transfer read, not on list items, and the list cannot filter on it.

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
| `RevertRequested`, `RevertAuthorized` | On a settled transfer: a revert (a new, linked transfer returning the funds) was requested, then authorized by everyone. The return is under way. |
| `RevertRejected`, `RevertCanceled` | The revert itself was rejected or canceled; the original settlement stands. |
| `Reverted`, `RevertedIncompleteAuthorization` | The revert settled: the funds came back. The second means it settled before every agent had authorized it. |

## An event did not arrive

1. Check the endpoint's message log under **Integration → Webhooks**: delivered, failed, or never sent.
2. Check the endpoint is enabled and subscribed to that event type.
3. Read the transfer with `GET /api/v1/transfers/{id}`. `requires_action: true` means something waits on you, whatever
   events you received.
