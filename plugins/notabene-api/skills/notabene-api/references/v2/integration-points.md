# Integration points (v2)

Where Notabene v2 hooks into an institution's own code: a pre-authorization step in the withdrawal flow, address
registration (relationships) when deposit addresses are issued, a one-off backfill of existing addresses, and a check
in the deposit crediting path. Find each place in the codebase before writing code. Endpoints, bodies, statuses and
webhooks are in [the Travel Rule API guide](guide-travel-rule-api.md), [the API reference](api-reference.md) and
[the webhook catalogue](webhooks.md); a wallet service acting for other institutions also reads
[the wallet service guide](wallet-service-guide.md). Pseudocode below is language-neutral.

## Contents

- [1. Find the integration points](#1-find-the-integration-points)
- [2. Withdrawals: a hook before sending](#2-withdrawals-a-hook-before-sending)
- [3. Register addresses as they are issued](#3-register-addresses-as-they-are-issued)
- [4. Upload existing addresses](#4-upload-existing-addresses)
- [5. Deposits: tie in when funds arrive](#5-deposits-tie-in-when-funds-arrive)
- [6. Checklist](#6-checklist)

## 1. Find the integration points

Search before you design. Useful terms (case-insensitive, across services, jobs and handlers):

| Integration point | Search for |
| --- | --- |
| Withdrawal flow | `withdraw`, `withdrawal`, `payout`, `send`, `transfer`, `outgoing`, `WithdrawalStatus`, `state machine`, `approve` |
| Sign / broadcast | `sign`, `broadcast`, `sendRawTransaction`, `submitTransaction`, `pushTx`, `createTransaction` on a custody SDK (Fireblocks, BitGo, …), `nonce` |
| Address creation | `createWallet`, `newAddress`, `generateAddress`, `deriveAddress`, `depositAddress`, `allocateAddress`, `assignAddress`, `xpub` |
| Deposit detection | `onDeposit`, `deposit`, `incoming`, `listener`, `watcher`, `indexer`, `confirmations`, `blockHeight`, custody callbacks (`webhook`, `callback`, `notification`) |
| Crediting | `credit`, `balance`, `ledger`, `journal`, `available`, `pending` |
| Async machinery | job queue / worker / scheduler, outbox, existing webhook routes and their signature checks |
| Existing Notabene code | `notabene`, `api.eu1.notabene.id`, `api.us1.notabene.id`, `/entities/`, `entityDID`, `svix` |

Answer these, with file and function names, before writing code. Ask the user only what the code cannot show:

- Which function signs or broadcasts? Is signing separate from broadcasting (a signed transaction held for later)?
- Up to which state is a withdrawal still cancellable, with the customer's balance returned?
- Where is a deposit credited to the customer, and is there already a "pending" or "held" state before credit?
- Which deposit addresses are the institution's own, and which does a custody provider (wallet service) hold?
- Is there a job queue for retries, and how are external webhooks received and verified?
- Does the institution act for other institutions (a wallet service)? Then follow
  [the wallet service guide](wallet-service-guide.md) for role detection and `custodian`.

## 2. Withdrawals: a hook before sending

Insert one state between "validated" (balance reserved, limits and screening passed) and "signed/broadcast":

```text
requested → validated → pending_travel_rule → ready_to_broadcast → broadcast → settle_reported → done
                              └→ cancelled (balance returned)
```

1. **Before the call**, persist a `ref` on the withdrawal: `^[a-zA-Z0-9_-]{1,255}$`, unique per transfer. A reused
   `ref` answers `409` (`Transfer with ref … already exists`) ([API reference](api-reference.md)).
2. **Create** with `POST /entities/{entityDID}/tx` **before** broadcasting: `originator`, `beneficiary`, CAIP-19
   `asset`, decimal-string `amount`, the `agents` chain and `ref`
   ([guide § Create](guide-travel-rule-api.md#1-create-a-transfer)). Store the transfer id.
3. **Park** the withdrawal in `pending_travel_rule`. Webhook handlers, not the request thread, move it on.
4. **Answer** `tap.requirePresentationRequested` by POSTing the IVMS101 PII to the payload's `callbackUrl`
   ([guide § Provide PII](guide-travel-rule-api.md#2-provide-pii-when-requested)).
5. **Resume** only on `notification.transferStatusChanged` with `toStatus: "AUTHORIZED"`. A per-agent
   `notification.transferAgentStatusChanged` is not enough.
6. **Stop** on `REJECTED`: do not broadcast; cancel and return the balance. `FLAGGED` is held for review: keep it
   parked. How the originator withdraws its own transfer is not spelled out in these guides; the reject reasons
   include `CANCELED_BY_INITIATOR`, which suggests `POST .../tx/{id}/reject`, but that is not confirmed.
7. **Broadcast** once (an atomic `pending → broadcasting` claim, since `AUTHORIZED` can arrive twice), then
   `POST /entities/{entityDID}/tx/{id}/settle` with `{ "settlementId": "eip155:{chainId}:tx/{txHash}" }`.

```text
onWithdrawalValidated(w):
  w.ref = w.ref ?? newRef(w); save(w)                                // persisted before any call
  enqueue(createTravelRule, w.id)

createTravelRule(w):                                                  // job, retried with backoff
  resp = POST /entities/{entityDID}/tx {ref: w.ref, …}
  if resp.status == 409: transfer = (GET /entities/{entityDID}/tx?refs=w.ref).data[0]   // earlier attempt landed
  w.state = pending_travel_rule; w.transferId = transfer.id; save(w)

onWebhook(event):                                                     // verified, de-duplicated on svix-id
  transfer = GET /entities/{entityDID}/tx/{event.payload.id}
  w = findWithdrawalByTransferId(transfer.id)
  if transfer.status == AUTHORIZED and claim(w, pending_travel_rule → ready_to_broadcast): enqueue(broadcast, w)
  if transfer.status == REJECTED: cancelAndRefund(w)
```

**Timeouts and outages.** Never broadcast because Notabene did not answer. Retry the create with the same `ref`; a
`409` means the first attempt landed, so look it up with `GET /entities/{entityDID}/tx?refs=<ref>`. Leave the
withdrawal parked while the API is unreachable. Webhooks are at least once and in any order: re-read the transfer
before acting ([guide § Delivery](guide-travel-rule-api.md#delivery-guarantees)). How long to wait for a
counterparty before cancelling is the institution's decision; the guides set no timeout. Entities with a settlement
policy for which Notabene enabled settlement signals also get `notification.settleTransaction`, whose `action`
(`SETTLE`, `REVIEW`, `CANCEL`, `ACK`) covers timeouts ([webhooks](webhooks.md#settlement-signal-event)).

## 3. Register addresses as they are issued

v2 has no customer registration: customers are IRIs (`mailto:…`, `did:…`) named on each transfer. Address
registration links the address to your institution, not to a customer. Hook the function that creates or allocates
a deposit address; after it is saved, enqueue a job that calls `POST /entities/{entityDID}/relationships`
([API reference § Relationships](api-reference.md#relationships)):

```json
{ "from": "did:pkh:eip155:1:0xYourDepositAddress", "to": "did:web:your-vasp.com" }
```

- `from` is the address as `did:pkh` (CAIP-10 inside), or a plain address with a CAIP-19 `asset` to normalize it.
- Add `"custodian": "<wallet service DID>"` when a custody provider holds the address for you. A wallet service
  registering a client's address sets `to` to the client institution and `custodian` to its own DID
  ([wallet service guide](wallet-service-guide.md)).
- A relationship naming your `entityDID` as `from`, `to` or `custodian` is confirmed on creation, so transfers to it
  don't wait for `tap.requireRelationshipConfirmationRequested`.
- **Never block wallet creation on it.** Mark the address `registration_pending`, return it, and let the job retry
  with backoff until `201`; then mark it `registered`.
- What a second `POST` for an existing relationship returns is not confirmed. Before retrying an unclear attempt,
  check `GET /entities/{entityDID}/relationships?from=…`. `PATCH /entities/{entityDID}/relationships?from=…&to=…` is
  documented as a confirm that creates the relationship if missing (upsert); whether it records `custodian` exactly
  as `POST` does is not confirmed.
- When an address is retired: `DELETE /entities/{entityDID}/relationships?from=…&to=…`.

## 4. Upload existing addresses

No bulk or upload endpoint for relationships appears in v2's route declarations: backfill with a loop of
`POST /entities/{entityDID}/relationships`, one address per call.

```text
backfill():
  cursor = loadCheckpoint()
  for batch in addresses where id > cursor order by id, page 100:
    for a in batch where a.state != registered:
      if GET /entities/{entityDID}/relationships?from=a.did is empty:
        POST /entities/{entityDID}/relationships {from: a.did, to: entityDID, custodian?: a.custodian}
      mark a registered
    saveCheckpoint(batch.lastId)
    sleep(pace)                          // no published rate limit; keep a modest, configurable rate
```

Run it first on a sandbox entity (same base URL as production in v2), spot-check with
`GET /entities/{entityDID}/relationships`, and keep step 3's hook live so new addresses never need another backfill.
No customers need registering first.

## 5. Deposits: tie in when funds arrive

**Travel Rule message first (usually the case).** Handle the incoming transfer in webhook handlers before the funds
land ([guide § Incoming](guide-travel-rule-api.md#incoming-transfers-beneficiary)):

1. `tap.requireRelationshipConfirmationRequested` (only for unregistered addresses): `PATCH` the
   `confirmCallbackUrl` if the address is yours (URL-encode its DIDs).
2. `tap.requirePresentationSatisfied`: `GET .../tx/{id}?decrypt=true`, match the beneficiary name against your KYC,
   complete the configured checks, then authorize (policies, or `POST .../tx/{id}/authorize`, optionally with
   `settlementAddress`) or reject (`BENEFICIARY_NOT_FOUND` on a name mismatch).
3. Record the transfer id against the expected deposit (settlement address, asset, amount) so the deposit listener
   links it.

**In the deposit listener**, after confirmations and before crediting:

```text
onDepositConfirmed(d):
  r = GET /entities/{entityDID}/tx/match?settlement_id=eip155:1:tx/<hash>&settlement_address=<address>
                                         &direction=INCOMING
  if r.meta.match_strategy == "none":
    POST /entities/{entityDID}/tx {ref: depositRef(d), settlementId, originator: {"@id": sender did:pkh}, …}
                                                                    // persisted ref; 409 = already created
  decide(d, transfer)

decide(d, transfer):
  AUTHORIZED / SETTLED              → credit
  REJECTED / FLAGGED                → hold; route to compliance (funds already arrived)
  AWAITING-YOURS / AWAITING-COUNTERPARTY → hold; finish the requirements
```

- `strict` is an exact match; `broadened` matched after dropping `settlement_id_index`: check amount and asset
  before linking it.
- Creating the unmatched deposit puts it in the compliance workflow; Notabene then tries to identify the originator
  institution and request the data ([guide § Reconcile](guide-travel-rule-api.md#4-reconcile-received-transfers)).
- When to credit is the institution's compliance policy; the guides do not define a credit rule. Ask the user; the
  pseudocode shows the conservative choice.

## 6. Checklist

- [ ] Each integration point named by file and function; ambiguities confirmed with the user.
- [ ] Withdrawals carry a persisted `ref` and a `pending_travel_rule` state before signing; `409` is recovered by
      looking the transfer up by `ref`.
- [ ] Broadcast only on transfer-level `AUTHORIZED`, claimed atomically; then `settle` with the CAIP-220 hash.
- [ ] Nothing broadcasts during a Notabene outage or timeout.
- [ ] Every new deposit address is registered as a relationship by a retried, non-blocking job (`custodian` where
      a custody provider holds it).
- [ ] Existing addresses backfilled with a resumable, paced loop.
- [ ] The deposit listener calls `tx/match`, creates the transfer when nothing matches, and credits or holds by
      status.
- [ ] Webhooks enabled (`PUT /entities/{entityDID}/webhook` plus a Svix portal endpoint), verified, idempotent.
