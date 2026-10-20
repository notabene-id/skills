# Deposits

Your customer is receiving crypto. When the sender's institution is on Notabene, a Travel Rule message arrives
before the funds and tells you who is sending; when it is not, Deposit Assist asks your customer, or lets them share
a link so the sender enters their own details. Your authorization rules then decide, the sender settles, you credit.

## Contents

- [Quick start](#quick-start)
- [How it works](#how-it-works)
- [1. Collect transfer details from your customer](#1-collect-transfer-details-from-your-customer)
- [Do this once: register your deposit addresses](#do-this-once-register-your-deposit-addresses)
- [With a Travel Rule message](#with-a-travel-rule-message)
- [Without a Travel Rule message](#without-a-travel-rule-message)
- [Address ownership requests](#address-ownership-requests)

## Quick start

When crypto is coming to one of your customers, you find out who it is from; your rules say yes or no.

1. Register each deposit address with `POST /api/v1/wallets` when you hand it to a customer ([Wallet
   addresses](wallets.md)).
2. Mount Deposit Assist in your deposit screen, with a customer token your backend mints and Counterparty Assist
   on, so a deposit without a Travel Rule message still gets the sender's details.
3. On `transfer.created.v3` with `direction: "Incoming"`, or when the component completes, read the transfer.
4. Your authorization rules check the beneficiary is your customer and authorize or reject; call
   `POST /api/v1/transfers/{id}/authorize` only for a decision they leave to you.
5. Credit your customer when the transfer reads `Settled`.

There are two flows. Work out which one you are in first:

- **With a Travel Rule message.** The sending institution creates the transfer before it broadcasts. Your rules
  decide; it settles. This is the common case between institutions on Notabene.
- **Without a Travel Rule message.** Funds arrive, or are expected, from a self-hosted wallet or an institution
  outside Notabene, with no data attached. Your customer, or the sender through a shared link, supplies the details;
  you create the record and Notabene finds the sender.

## How it works

Each step lines up with one call your code makes, one event it handles, or a decision your authorization rules
make for you.

### With a Travel Rule message

The sending institution drives the transfer and your code answers.

### Without a Travel Rule message

Nobody announced the transfer, so your side supplies what the message would have carried, and your rules decide
on that.

## 1. Collect transfer details from your customer

A deposit with a Travel Rule message arrives with the sender's details. One without, from a self-hosted wallet or
an institution outside Notabene, does not: Deposit Assist, from Notabene's JavaScript SDK, asks your customer who is
sending in your deposit screen, and with Counterparty Assist on lets them share a link so the sender enters their
own details and, for the customer's own wallet, proves they hold it. It hands your backend the transfer to post.
[Embedded components](transact-embedded.md) covers every option; this is the shortest path.

**Backend: mint two tokens.** Both for the customer who is signed in, never for an id the browser sends. The
customer token lets the component work for that customer; the Counterparty Assist token, minted per link the
customer shares, lets the sender fill in their part. Both are read-only.

```bash
curl -s https://sandbox.notabene.id/api/v1/entities/$YOUR_DID/delegate-token \
  -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"customer_ref":"urn:uuid:0c8f6a52-6e83-4f2b-9d0a-7b1f0e7c2a41"}'

curl -s https://sandbox.notabene.id/api/v1/entities/$YOUR_DID/counterparty-assist-token \
  -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"customer_ref":"urn:uuid:0c8f6a52-6e83-4f2b-9d0a-7b1f0e7c2a41"}'
```

`$YOUR_DID` is your institution's DID (from `GET /api/v1/identity`). Each response carries `access_token`,
`expires_in` (about an hour) and `jti`; keep the Counterparty Assist token's `jti` to revoke the link afterwards.
Put each call behind an endpoint only your signed-in customer can reach.

**Front end: mount the component.** Install `@notabene/javascript-sdk` (2.24 or later), create the v2.1
`Notabene` with both token callbacks, and mount Deposit Assist in your deposit screen with
`counterpartyAssist` on. `nodeUrl` is the base URL your backend calls.

```js
import { Notabene, getRefreshResult } from '@notabene/javascript-sdk/src/v21';

const notabene = new Notabene({
  nodeUrl: 'https://sandbox.notabene.id',
  locale: 'en',
  fetchToken: async () => {
    const res = await fetch('/api/notabene/customer-token', { method: 'POST' });
    const { access_token } = await res.json();
    return access_token;
  },
  fetchCounterpartyAssistToken: async () => {
    const res = await fetch('/api/notabene/counterparty-assist-token', { method: 'POST' });
    const { access_token, jti } = await res.json();
    return { token: access_token, jti };
  },
});

const deposit = notabene.createDepositAssist(
  { asset: 'USDC', amountDecimal: 250, source: '0x6b175474e89094c44da98b954eedeac495271d0f' },
  { counterpartyAssist: { counterpartyTypes: ['natural', 'legal', 'self'] } },
);
deposit.mount('#nb-deposit');

const result = await deposit.completion();
const { transfer } = 'refreshSource' in result
  ? await waitForCounterparty(result.refreshSource) // the sender finishes on their own time
  : result;
await fetch('/api/deposits', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ transfer }),
});

async function waitForCounterparty(refreshSource) {
  for (;;) {
    const state = await getRefreshResult(refreshSource);
    if (state.status === 'completed') return state.result;
    await new Promise((r) => setTimeout(r, 5000));
  }
}
```

Leave `source` out when the sender's address cannot be known (Lightning, Monero, a batched exchange withdrawal):
the component then offers to pick the sending institution, or a self-declaration for the customer's own wallet.
When the customer shares a link, the component completes with `{ refreshSource }` instead of a transfer;
`getRefreshResult` reads the sender's `{ transfer }` once they are done, however long that takes, so a backend job
can do the waiting instead of the page. Revoke the link's token afterwards with
`POST /api/v1/entities/{your DID}/delegate-token/{jti}/revoke`.

**Server-only instead?** If no web front end exists, collect the sender's name and institution in your own deposit
form and build the `POST /api/v1/transfers` body yourself, as under *Without a Travel Rule message* below.

## Do this once: register your deposit addresses

Register every deposit address as you issue it, with the customer it belongs to ([Wallet addresses](wallets.md)).
Senders then find you from the address, and Notabene confirms the address is yours without asking you.

## With a Travel Rule message

1. **Notice it.** `transfer.created.v3` arrives with `direction: "Incoming"`, then `requirements.pending.v3`. Read the
   full transfer with `GET /api/v1/transfers/{id}`.
2. **Let your rules answer the requirements.** Your authorization rules, under
   **Authorization Rules → Policies**, resolve what they can: whether the name matches your customer, screening,
   your limits. Your code answers only what is routed to it, by requirements whose `agentId` is your DID:
   - `wallet-address-confirmation` — is the address yours? Resolve with `outcome: "confirmed"` or `"rejected"`. Not
     asked for addresses you registered.
   - `beneficiary-name-match` — when your rules leave the match to your system: resolve with `matched`,
     `partial_match` (holds the transfer for your compliance team) or `mismatch` (final).
   - Travel Rule data owed back → `POST /api/v1/transfers/{id}/send-presentation`.
3. **Your rules decide.** When every rule clears, your agent is `Authorized` and the transfer moves on without a
   call from you; a transfer your rules flag waits for your compliance team in the dashboard. Authorize from code
   only for a decision your rules leave to it, with the address the funds should land on:

   ```bash
   curl -s https://sandbox.notabene.id/api/v1/transfers/$ID/authorize \
     -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
     -d '{"settlement_address":"eip155:1:0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984"}'
   ```

   `settlement_address` must be CAIP-10 here (chain prefix included), unlike on create. Or
   `POST /api/v1/transfers/{id}/reject` with a `reason`. A `409` on authorize means the transfer changed under you:
   re-read it and retry.
4. **Credit.** The sender settles; you receive `transfer.status_changed.v3` with `toStatus: "Settled"` and a
   `settlementId`. Credit the customer once your own chain monitoring confirms that transaction.

## Without a Travel Rule message

1. **Create the record** from what Deposit Assist collected, or from your deposit listener, with
   `POST /api/v1/transfers`. `transfer` from the component already names you as the beneficiary's institution
   (`beneficiary_entity_did`, your DID) and carries the sender's details; that is what makes this a deposit
   rather than a withdrawal. Add `ref`, your customer's id as `beneficiary_id`, and `settlement_id` once the funds
   have arrived.

   ```json
   {
     "ref": "a3c1d9b0-4f7e-4f1e-8a51-0d2b6c9e7f13",
     "asset": "USDC",
     "amount": "250.00",
     "beneficiary_entity_did": "did:web:your-institution.example",
     "beneficiary_id": "urn:uuid:0c8f6a52-6e83-4f2b-9d0a-7b1f0e7c2a41",
     "originator_id": "urn:uuid:3b9d2c70-5e1f-4a8b-9c6d-1e2f3a4b5c6d",
     "originator": {"ivms": {"naturalPerson": {"name": {"nameIdentifier": [
       {"primaryIdentifier": "Jones", "secondaryIdentifier": "Bob", "naturalPersonNameIdentifierType": "LEGL"}]}}}},
     "settlement_id": "0x9fc76417374aa880d4449a1f7f31ec597f00b1f6f3dd2d66f4c9c6c445836d8b",
     "source_address": "0x6b175474e89094c44da98b954eedeac495271d0f",
     "settlement_address": "0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984"
   }
   ```

   The sending institution is discovered from `source_address`; the component sets `originator_entity_did` when it
   found one, and a `settlement_id` is then required. A self-hosted sender's proof travels as
   `originator_proof_of_ownership`.
2. **Your rules decide, as above.** A transfer created with `settlement_id` starts as
   `SettledIncompleteAuthorization`: the funds moved, decisions are outstanding. Your authorization rules work
   through them; your code answers only what is routed to it. It is not finished until it reads `Settled`.

Done when the transfer reads `Settled` and your ledger entry carries its `settlement_id`.

## Address ownership requests

`relationship.confirmation_requested.v3` asks you to confirm that an address on a transfer belongs to one of your
customers. Answer with `POST /api/v1/transfers/{id}/requirements/wallet-address-confirmation/resolve` and
`{"outcome":"confirmed"}` or `"rejected"` ([Wallet addresses](wallets.md#when-someone-asks-whether-an-address-is-yours)).
Registering your addresses as you issue them means most of these never reach you.
