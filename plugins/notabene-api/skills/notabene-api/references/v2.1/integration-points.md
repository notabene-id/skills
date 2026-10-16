# Integration points (v2.1)

Where Notabene v2.1 hooks into an institution's own code: a pre-authorization step in the withdrawal flow, customer
and address registration when customers onboard and deposit addresses are issued, a one-off backfill of existing ones,
and a check in the deposit crediting path. Find each place in the codebase before writing code. Endpoints, bodies,
events and statuses are in [Send a transfer](transact-send.md), [Receive a transfer](transact-receive.md),
[Customers](customers.md), [Wallet addresses](wallets.md) and [Webhooks](webhooks.md); the API reference
(`/api/openapi.json`) is authoritative for field names. Pseudocode below is language-neutral.

## Contents

- [1. Find the integration points](#1-find-the-integration-points)
- [2. Withdrawals: a hook before sending](#2-withdrawals-a-hook-before-sending)
- [3. Register customers and addresses as they are issued](#3-register-customers-and-addresses-as-they-are-issued)
- [4. Upload existing customers and addresses](#4-upload-existing-customers-and-addresses)
- [5. Deposits: tie in when funds arrive](#5-deposits-tie-in-when-funds-arrive)
- [6. Checklist](#6-checklist)

## 1. Find the integration points

Search before you design. Useful terms (case-insensitive, across services, jobs and handlers):

| Integration point | Search for |
| --- | --- |
| Withdrawal flow | `withdraw`, `withdrawal`, `payout`, `send`, `transfer`, `outgoing`, `WithdrawalStatus`, `state machine`, `approve` |
| Sign / broadcast | `sign`, `broadcast`, `sendRawTransaction`, `submitTransaction`, `pushTx`, `createTransaction` on a custody SDK (Fireblocks, BitGo, …), `nonce` |
| Customer onboarding | `onboard`, `signup`, `kyc`, `createUser`, `createCustomer`, `verified` |
| Address creation | `createWallet`, `newAddress`, `generateAddress`, `deriveAddress`, `depositAddress`, `allocateAddress`, `assignAddress`, `xpub` |
| Deposit detection | `onDeposit`, `deposit`, `incoming`, `listener`, `watcher`, `indexer`, `confirmations`, `blockHeight`, custody callbacks (`webhook`, `callback`, `notification`) |
| Crediting | `credit`, `balance`, `ledger`, `journal`, `available`, `pending` |
| Async machinery | job queue / worker / scheduler, outbox, existing webhook routes and their signature checks |
| Existing Notabene code | `notabene`, `sandbox.notabene.id`, `eu-a.notabene.id`, `us-a.notabene.id`, `/api/v1/transfers`, `svix` |

Answer these, with file and function names, before writing code. Ask the user only what the code cannot show:

- Which function signs or broadcasts? Is signing separate from broadcasting (a signed transaction held for later)?
- Up to which state is a withdrawal still cancellable, with the customer's balance returned?
- Where is a deposit credited to the customer, and is there already a "pending" or "held" state before credit?
- Where does a customer get a stable id that can become their Notabene customer id (a `urn:uuid:`)?
- Which deposit addresses belong to one customer, which are shared (omnibus), and does a connected wallet provider
  hold them? A connected provider changes the integration type ([Go live](go-live.md)).
- Is there a job queue for retries, and how are external webhooks received and verified?

## 2. Withdrawals: a hook before sending

Insert one state between "validated" (balance reserved, limits and screening passed) and "signed/broadcast":

```text
requested → validated → pending_travel_rule → ready_to_broadcast → broadcast → settle_reported → done
                              └→ cancelled (balance returned)
```

1. **Optionally assess** with `POST /api/v1/transfers/assess` while the customer confirms, to learn which fields the
   form must collect ([Send § Assess](transact-send.md#1-assess-optional-recommended)).
2. **Before the call**, persist a UUID `ref` on the withdrawal. Creation is idempotent on `ref`: replaying the
   identical body returns the existing transfer; the same `ref` with a different body answers `409`.
3. **Create** with `POST /api/v1/transfers`: `ref`, `asset`, whole-unit decimal `amount`, `settlement_address`,
   `originator_id`, and `beneficiary_id` with the beneficiary's details inline
   ([Send § Create](transact-send.md#2-create-the-transfer)). Store `transferId`; the status is `Active`.
4. **Park** the withdrawal in `pending_travel_rule`. Webhook handlers, not the request thread, move it on. Answer
   `requirements.pending.v3` items whose `agentId` is your DID
   ([Send § Answer requirements](transact-send.md#3-answer-requirements)).
5. **Resume** only on `transfer.status_changed.v3` with `toStatus: "ReadyToSettle"`.
6. **Stop** on `Rejected` or `Canceled`: do not broadcast; return the balance. If the customer abandons first,
   `POST /api/v1/transfers/{id}/cancel` (withdraws a transfer you initiated, before it settles).
7. **Broadcast** once (an atomic claim, since events repeat), then `POST /api/v1/transfers/{id}/settle` with
   `{ "settlement_id": "<hash>" }`. Done when the transfer reads `Settled`.

```text
onWithdrawalValidated(w):
  w.ref = w.ref ?? uuid(); save(w)                                    // persisted before any call
  enqueue(createTravelRule, w.id)

createTravelRule(w):                                                  // job, retried with backoff
  t = POST /api/v1/transfers {ref: w.ref, …same body every attempt…}
  w.state = pending_travel_rule; w.transferId = t.transferId; save(w)

onWebhook(event):                                                     // raw-body Svix check, de-dup on svix-id
  t = GET /api/v1/transfers/{event.payload.id}
  w = findWithdrawalByTransferId(t.id)
  if t.status == ReadyToSettle and claim(w, pending_travel_rule → ready_to_broadcast): enqueue(broadcast, w)
  if t.status in (Rejected, Canceled): cancelAndRefund(w)
```

**Timeouts and outages.** Never broadcast because Notabene did not answer. Retry the create with the identical body
and `ref`; `5xx` means retry with backoff ([Authentication § Conventions](authentication.md#conventions)). Leave the
withdrawal parked. `Rejected` can still be lifted by re-evaluation, so re-read before refunding if your flow allows a
late resume. A daily sweep of `GET /api/v1/transfers?status=Active` (reading each for `requires_action`) catches
events that never arrived ([Webhooks § Handle events safely](webhooks.md#3-handle-events-safely)). How long to wait
for a counterparty before cancelling is the institution's decision; the guides set no timeout.

## 3. Register customers and addresses as they are issued

**Customers first.** Hook onboarding (or KYC completion): `POST /api/v1/customers` with your stable `id`, `type`,
`jurisdiction` and `profile.ivms` ([Customers](customers.md)). `409` means already registered: treat it as done. A
wallet's `customer_id` must already exist.

**Then each deposit address.** Hook the function that creates or allocates it; after it is saved, enqueue a job that
calls `POST /api/v1/wallets` ([Wallet addresses § Register](wallets.md#register-an-address)):

```json
{ "address": "eip155:1:0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984",
  "customer_id": "urn:uuid:0c8f6a52-6e83-4f2b-9d0a-7b1f0e7c2a41" }
```

- `address` is CAIP-10 only (chain prefix required). Omit `customer_id` for an omnibus address. Set `custodian` to
  the provider's DID when a wallet provider holds the address.
- Registering again with the same content succeeds; different content for the same address answers `409`. A `409`
  is a data conflict to investigate (another customer or custodian on record), not a success.
- **Never block wallet creation on it.** Mark the address `registration_pending`, return it, and let the job retry
  with backoff; order it after the customer's registration job.
- Registering also answers confirmations already waiting on live transfers to that address.
- When an address is retired: `DELETE /api/v1/wallets/{address}`.

## 4. Upload existing customers and addresses

The API reference has no bulk or upload route for wallets or customers: `POST /api/v1/wallets` and
`POST /api/v1/customers` take one record each. Backfill with a resumable, paced loop, customers before their
addresses:

```text
backfill():
  cursor = loadCheckpoint()
  for batch in customers where id > cursor order by id, page 100:
    for c in batch: POST /api/v1/customers {…}        // 201 or 409 = done
    for a in addressesOf(batch):
      POST /api/v1/wallets {address: caip10(a), customer_id?: a.customerId, custodian?: a.custodian}
                                                      // 201 = done; 409 = record a conflict, continue
    saveCheckpoint(batch.lastId)
    sleep(pace)                                       // no published rate limit; keep a modest, configurable rate
```

Retry `5xx` and timeouts with backoff by replaying the same body. Run it in sandbox first, then again in production
(they share nothing). Spot-check with `GET /api/v1/wallets` (cursor-paged, `limit` up to 200) and report the `409`
conflicts to the user. Keep step 3's hooks live so new records never need another backfill.

## 5. Deposits: tie in when funds arrive

**Announced before settlement (the common case).** The Travel Rule data arrives before the funds
([Receive § Announced](transact-receive.md#announced-before-settlement)):

1. `transfer.created.v3` with `direction: "Incoming"`: read `GET /api/v1/transfers/{id}`; record the transfer id
   against the expected deposit (settlement address, asset, amount).
2. `requirements.pending.v3`: resolve the items whose `agentId` is your DID (`wallet-address-confirmation`,
   `beneficiary-name-match`; `POST /api/v1/transfers/{id}/send-presentation` when Travel Rule data is owed back).
3. Authorize with a CAIP-10 `settlement_address`, or reject with a `reason`. A `409` on authorize: re-read, retry
   once.
4. `transfer.status_changed.v3` → `Settled` carries the `settlementId`: credit once your own chain monitoring confirms
   that transaction.

**In the deposit listener**, after confirmations and before crediting:

```text
onDepositConfirmed(d):
  r = GET /api/v1/transfers/match?settlement_id=<hash>&settlement_address=<CAIP-10>[&source_address=…&asset=…]
  if r.data is empty:
    POST /api/v1/transfers {ref: persistedRef(d), asset, amount, beneficiary_entity_did: <your DID>,
                            beneficiary_id, settlement_id, source_address, settlement_address}
  decide(d, transfer)

decide(d, transfer):
  Settled                                   → credit
  SettledIncompleteAuthorization            → hold; keep answering requirements
  Rejected / SettledCanceled                → hold; route to compliance (funds already arrived)
  Active / ReadyToSettle                    → hold until Settled
```

- `match` returns `data[]`, `all_filters_applied` and `dropped_filters`. When `all_filters_applied` is `false`, the
  rows are candidates that failed part of the query: don't auto-link them.
- The unmatched create is "seen on-chain first": your DID from `GET /api/v1/identity` as `beneficiary_entity_did`;
  the sender is discovered from `source_address`
  ([Receive § Seen on-chain first](transact-receive.md#seen-on-chain-first)).
  It starts as `SettledIncompleteAuthorization` and is finished only at `Settled`.
- Calling `match` before creating is this guide's recommendation to avoid a second record for an announced
  transfer; the receive guide itself goes straight to create.
- The guides say to credit on `Settled`; holding funds before that is the institution's compliance policy. Ask the
  user.

## 6. Checklist

- [ ] Each integration point named by file and function; ambiguities confirmed with the user.
- [ ] Withdrawals carry a persisted UUID `ref` and a `pending_travel_rule` state before signing; retries replay the
      identical body.
- [ ] Broadcast only on `ReadyToSettle`, claimed atomically; then `settle` with the hash.
- [ ] Nothing broadcasts during a Notabene outage or timeout; a daily sweep catches missed events.
- [ ] Customers registered at onboarding (`409` = done) before their addresses.
- [ ] Every new deposit address registered in CAIP-10 by a retried, non-blocking job (`custodian` where a provider
      holds it).
- [ ] Existing customers and addresses backfilled with a resumable, paced loop; `409` wallet conflicts reported.
- [ ] The deposit listener calls `transfers/match`, creates the transfer when nothing matches, and credits on
      `Settled`.
- [ ] Walk [the pre-production checklist](pre-production.md) before live traffic.
