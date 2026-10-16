# Integration points (v1)

Where Notabene v1 hooks into an institution's own code: a pre-authorization step in the withdrawal flow, address
registration when deposit addresses are issued, a one-off backfill of existing addresses, and a check in the deposit
crediting path. Find each place in the codebase before writing code. Endpoints, bodies and statuses are in
[Send](send.md), [Receive](receive.md), [Webhooks](webhooks.md) and [Getting started](getting-started.md); the
facts that could not be confirmed are in [Caveats](caveats.md). Pseudocode below is language-neutral.

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
| Existing Notabene code | `notabene`, `api.notabene.id`, `/tx/create`, `vaspDID`, `transactionRef`, `svix` |

Answer these, with file and function names, before writing code. Ask the user only what the code cannot show:

- Which function signs or broadcasts? Is signing separate from broadcasting (a signed transaction held for later)?
- Up to which state is a withdrawal still cancellable, with the customer's balance returned?
- Where is a deposit credited to the customer, and is there already a "pending" or "held" state before credit?
- Which deposit addresses map to one customer, and which are shared (omnibus)?
- Is there a job queue for retries, and how are external webhooks received and verified?
- Which institution DID (`vaspDID`) and which `statusToProcessBlockchain` setting does the institution use?

## 2. Withdrawals: a hook before sending

Insert one state between "validated" (balance reserved, limits and screening passed) and "signed/broadcast":

```text
requested → validated → pending_travel_rule → ready_to_broadcast → broadcast → hash_reported → done
                              └→ cancelled (balance returned)
```

1. **Before the call**, persist a unique `transactionRef` on the withdrawal (for example `withdrawal-<id>`). It is
   v1's idempotency handle: a create repeating the `transactionRef` for the same `originatorVASPdid` returns the
   existing transaction instead of a new one ([Caveats](caveats.md)).
2. **Create** with `POST /tx/create` ([Send § 2](send.md#2-create)): amount in integer base units, asset symbol,
   `transactionBlockchainInfo.destination`, both customers' IVMS 101. Store the returned `id`. If it comes back
   `NEW`, `POST /tx/approve?id=<id>` unless an auto-approve rule already moved it to `SENT`.
3. **Park** the withdrawal in `pending_travel_rule`. The webhook handler, not the request thread, resumes it.
4. **Resume** on the safe-to-send signal ([README](README.md)):
   - `notification.processBlockchainTransaction`, fired when the transfer reaches the institution's
     `statusToProcessBlockchain` (usually `ACCEPTED`; read it with `GET /tf/vasps/{vaspDID}/getSettings`).
   - Without that setting there is no go-ahead event: resume on `notification.transactionUpdated` with
     `status: "ACCEPTED"`.
   - A create that returns `SAVED` (below threshold, self-hosted wallet, internal) is never sent to a counterparty
     and nothing will arrive: move straight to `ready_to_broadcast`.
5. **Stop** on `notification.haltBlockchainTransaction`, or on `REJECTED` / `DECLINED` / `CANCELLED`: do not
   broadcast; move to `cancelled` and return the balance. If the customer abandons first, `POST /tx/cancel` while
   the transaction is still cancellable ([Send § 6](send.md#6-cancel)).
6. **Broadcast**, then record the hash with `POST /tx/update { "id", "txHash" }` and re-read with
   `GET /tx/info?id=…` to verify: a `200` from `/tx/update` can be a silent no-op.

```text
onWithdrawalValidated(w):
  w.transactionRef = w.transactionRef ?? newRef(w); save(w)          // persisted before any call
  enqueue(createTravelRule, w.id)

createTravelRule(w):                                                  // job, retried with backoff
  tx = findByRef(w.transactionRef) ?? POST /tx/create {transactionRef: w.transactionRef, …}
  if tx.status == NEW: POST /tx/approve?id=tx.id
  if tx.status == SAVED: markReadyToBroadcast(w)
  else: w.state = pending_travel_rule; w.notabeneId = tx.id; save(w)

onWebhook(event):                                                     // verified, idempotent
  tx = GET /tx/info?id=event.payload.transaction.id
  w = findWithdrawalByRef(event.payload.transaction.transactionRef)
  if event.message == "notification.haltBlockchainTransaction" or tx.status in (REJECTED, DECLINED, CANCELLED):
    cancelAndRefund(w)
  else if event.message == "notification.processBlockchainTransaction" (or ACCEPTED when no setting):
    markReadyToBroadcast(w)                                           // atomic: only one worker broadcasts
```

**Timeouts and outages.** Never broadcast because Notabene did not answer. A create that timed out may still have
succeeded: retry with the same `transactionRef`, which returns the existing transaction ([Caveats](caveats.md)). Retry the job with backoff; leave the withdrawal parked. Webhooks are at least once and
unordered: key on `id` + `status` and never move your state backwards ([Webhooks](webhooks.md)). How long to wait for
a counterparty before cancelling is the institution's decision; the guides set no timeout.

Test the go-ahead path end to end on `api.notabene.dev` before relying on it: check your `statusToProcessBlockchain`
setting and that `processBlockchainTransaction` and `haltBlockchainTransaction` reach your handler ([Webhooks](webhooks.md)).

## 3. Register addresses as they are issued

Hook the function that creates or allocates a deposit address. After the address is saved, enqueue a job that
calls `POST /address/registerCustomer`
([Receive § Address registration](receive.md#address-registration-and-ownership)):

```json
{ "vaspDID": "did:ethr:0xbbbb…", "asset": "ETH", "address": "0xDepositAddress", "customerRef": "cust-123" }
```

- `vaspDID`, `address` and `customerRef` are required; `customerRef` is your id for the customer. It returns `200`
  with no body.
- Registration lets Notabene route incoming transactions to you and lets autoConfirm move them `SENT → ACK`.
- **Never block wallet creation on it.** Mark the address `registration_pending`, return it to the customer, and let
  the job retry with backoff until it gets a `2xx`; then mark it `registered`.
- Whether repeating the call for an already-registered address is a no-op or an error is not confirmed. Make the job
  idempotent on your side (skip addresses already marked `registered`), and check
  `GET /v1/addresses?vasp_did=…&source=ADDRESS_GRAPH&source_type=INTERNAL` if a retry's outcome is unclear.
- How to register a shared (omnibus) address without a `customerRef` is not confirmed; `customerRef` is required.
- When an address is retired: `DELETE /v1/addresses?vasp_did=…&addresses=<a1>,<a2>`.

## 4. Upload existing addresses

A one-off, resumable backfill of addresses issued before the integration. Two options:

**Bulk CSV upload.** `GET /address/upload/{vaspDID}` returns a short-lived presigned `{ uploadUrl, uploadFields[] }`
(each field a `{ key, value }`). POST the CSV to `uploadUrl` as `multipart/form-data`, sending every
`uploadFields` entry verbatim as a form field, plus the file. Not confirmed from the client code: the CSV's columns
and header row, the name of the file form field (presigned S3 POSTs conventionally take it as the last field,
`file`), size limits, and whether or when uploaded rows appear in `GET /v1/addresses`. Ask Notabene for the CSV
template before using this route.

**Loop over `/address/registerCustomer`.** Confirmed shape, one address per call:

```text
backfill():
  cursor = loadCheckpoint()
  for batch in addresses where id > cursor order by id, page 100:
    for a in batch where a.state != registered:
      POST /address/registerCustomer {vaspDID, asset: a.asset, address: a.address, customerRef: a.customerRef}
      mark a registered
    saveCheckpoint(batch.lastId)
    sleep(pace)                          // no published rate limit; keep a modest, configurable rate
```

Either way: run it once against `api.notabene.dev`, page `GET /v1/addresses?vasp_did=…&source=ADDRESS_GRAPH` to
spot-check the result, and keep step 3's hook live so new addresses never need another backfill.

## 5. Deposits: tie in when funds arrive

Two shapes reach the deposit path ([Receive](receive.md)):

**Travel Rule data first (the common case).** A `notification.transactionUpdated` arrives at `SENT` with your DID as
`beneficiaryVASPdid` before the funds. In the webhook handler: check the destination is your customer's address,
`POST /tx/confirm?id=…` (→ `ACK`; autoConfirm may already have done it), re-read `GET /tx/info` for the originator
PII, screen it, then `POST /tx/accept?id=…` or `POST /tx/decline`. Record the transaction id against the expected
deposit (destination, asset, amount) so the deposit listener finds it.

**Funds first.** In the deposit listener, after confirmations and before crediting:

```text
onDepositConfirmed(d):
  txs = GET /tx/query?txHash=d.hash&destination=d.address          // returns an array
  if txs is empty:
    POST /tx/notify {txHash, transactionAsset, transactionAmount (base units), beneficiaryVASPdid, destination,
                     origin, customerRef}                          // persist "notified" first; retry is a job
  decide(d, txs)

decide(d, tx):
  ACCEPTED                    → credit
  REJECTED / DECLINED         → hold; route to compliance (funds already arrived)
  INCOMPLETE / SENT / ACK     → hold; finish the transaction (let the originator update first, then accept)
```

- `/tx/notify` also links to a matching existing transaction and auto-confirms it; with no match it creates one at
  `INCOMPLETE`. Let the originator institution `/tx/update` first: a full beneficiary update before it closes the
  originator's window ([Receive § Deposit first](receive.md#deposit-first-txnotify)).
- Retrying `/tx/notify` after a timeout can create a second transaction (not confirmed either way): query with
  `/tx/query` before retrying.
- Whether to credit before `ACCEPTED` (and hold back only on rejection) is the institution's compliance policy, not
  something v1 enforces. Ask the user; the pseudocode above shows the conservative choice.
- Below-threshold and other `SAVED` transactions carry no decision: the institution's policy decides whether they
  are credited at once.

## 6. Checklist

- [ ] Each integration point named by file and function; ambiguities confirmed with the user.
- [ ] Withdrawals carry a persisted `transactionRef` and a `pending_travel_rule` state before signing.
- [ ] Broadcast only on `processBlockchainTransaction` (or `ACCEPTED` without the setting, or at once on `SAVED`);
      never after `haltBlockchainTransaction`; the hash is reported with `/tx/update` and verified.
- [ ] A timed-out create is looked up by `transactionRef` before retrying; nothing broadcasts during an outage.
- [ ] Every new deposit address is registered by a retried, non-blocking job.
- [ ] Existing addresses backfilled (CSV template confirmed with Notabene, or the registerCustomer loop).
- [ ] The deposit listener queries `/tx/query`, notifies when nothing matches, and credits or holds by status.
- [ ] Webhooks verified with Svix, idempotent, and re-read with `/tx/info` ([Webhooks](webhooks.md)).
