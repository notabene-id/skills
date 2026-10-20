# Withdrawals

Your customer is withdrawing crypto to an address outside your institution. Notabene's Withdrawal Assist component
collects the beneficiary's details from them in your app, Notabene Transact works out whether the Travel Rule
applies, finds the beneficiary's institution, exchanges the required data with it, and tells you when it is safe to
broadcast. Your code mints a customer token, posts the transfer and reacts to two events.

## Contents

- [Quick start](#quick-start)
- [How it works](#how-it-works)
- [1. Collect transfer details from your customer](#1-collect-transfer-details-from-your-customer)
- [2. Create the transfer](#2-create-the-transfer)
- [3. Answer requirements](#3-answer-requirements)
- [4. Wait for `ReadyToSettle`](#4-wait-for-readytosettle)
- [5. Broadcast, then settle](#5-broadcast-then-settle)
- [When things change](#when-things-change)

## Quick start

Before you send a customer's crypto out, tell Notabene, and wait for its go-ahead.

1. Mount Withdrawal Assist in your withdrawal form, with a customer token your backend mints; it completes with the
   `transfer` to post.
2. `POST /api/v1/transfers` with that `transfer`, your stored `ref` and your customer's id.
3. Wait for the `transfer.status_changed.v3` webhook with `toStatus: "ReadyToSettle"`.
4. Broadcast on-chain as you do today.
5. `POST /api/v1/transfers/{id}/settle` with the transaction hash.

Most transfers need nothing else. The steps below add the cases that need you.

## How it works

Each step between the two institutions lines up with one call your code makes, one event it handles, or a
decision your authorization rules make for you.

## 1. Collect transfer details from your customer

The Travel Rule data comes from your customer: who they are paying, which institution or self-hosted wallet holds
the destination, and proof that they own that wallet. Withdrawal Assist, from Notabene's JavaScript SDK, collects
all of it in your withdrawal form, in your customer's language, and hands your backend the transfer to post.
[Embedded components](transact-embedded.md) covers every option; this is the shortest path.

**Backend: mint a customer token.** The customer is registered ([Customers](customers.md)); mint for the one who
is signed in, never for an id the browser sends. The token is read-only and bound to that customer: it lets the
component work for them and nothing else.

```bash
curl -s https://sandbox.notabene.id/api/v1/entities/$YOUR_DID/delegate-token \
  -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"customer_ref":"urn:uuid:0c8f6a52-6e83-4f2b-9d0a-7b1f0e7c2a41"}'
```

`$YOUR_DID` is your institution's DID (from `GET /api/v1/identity`). The response carries `access_token`,
`expires_in` (about an hour) and `jti`. Put the call behind an endpoint only your signed-in customer can reach.

**Front end: mount the component.** Install `@notabene/javascript-sdk` (2.24 or later), create the v2.1
`Notabene` with a `fetchToken` that asks that endpoint, and mount Withdrawal Assist where the destination and
amount are entered. `nodeUrl` is the base URL your backend calls.

```js
import { Notabene } from '@notabene/javascript-sdk/src/v21';

const notabene = new Notabene({
  nodeUrl: 'https://sandbox.notabene.id',
  locale: 'en',
  fetchToken: async () => {
    const res = await fetch('/api/notabene/customer-token', { method: 'POST' });
    const { access_token } = await res.json();
    return access_token;
  },
});

const withdrawal = notabene.createWithdrawalAssist({
  asset: 'USDC',
  amountDecimal: 250,
  destination: '0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984',
});
withdrawal.mount('#nb-withdrawal');
withdrawal.update({ amountDecimal: 300 }); // as the customer edits the form

const { transfer } = await withdrawal.completion();
await fetch('/api/withdrawals', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ transfer }),
});
```

Every field is optional: the component asks the customer for whatever you leave out. It finds the institution
behind the destination, asks for the beneficiary's name and whatever else the jurisdictions involved require, and
collects an ownership proof when the destination is the customer's own wallet. Give it
`counterpartyAssist: { counterpartyTypes: ['natural', 'legal', 'self'] }` and a `fetchCounterpartyAssistToken`
to let your customer share a link so the beneficiary enters their own details
([Counterparty Assist](transact-embedded.md#counterparty-assist-optional)).

**Server-only instead?** If no web front end exists, `POST /api/v1/transfers/assess` with the asset, amount and
`destinationAddress` tells you whether the Travel Rule applies and which personal-data fields are `required`, so
your own form collects those and nothing more.

## 2. Create the transfer

`transfer` is the `POST /api/v1/transfers` body the component filled in. Add `ref`, your customer's id as
`originator_id`, and post it with your own credential:

```bash
curl -s https://sandbox.notabene.id/api/v1/transfers \
  -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{
    "ref": "5f0c7d1e-1b8e-4a4e-9d43-2f7e6b0a9c11",
    "asset": "USDC",
    "amount": "250.00",
    "settlement_address": "0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984",
    "originator_id": "urn:uuid:0c8f6a52-6e83-4f2b-9d0a-7b1f0e7c2a41",
    "beneficiary_id": "urn:uuid:7d2e4b10-93a5-4c8e-b1f6-2a9c0d5e8f34",
    "beneficiary": {"ivms": {"naturalPerson": {"name": {"nameIdentifier": [
      {"primaryIdentifier": "Jones", "secondaryIdentifier": "Bob", "naturalPersonNameIdentifierType": "LEGL"}]}}}}
  }'
```

The response is `{"transferId": "…", "status": "Active"}`. Store `transferId` on your withdrawal record.

- **`ref`** is your idempotency key. Generate and persist it before the call; on a timeout, replay the identical body.
- **The receiving institution** is discovered from `settlement_address`. The component sets
  `beneficiary_entity_did` when it found it; set it yourself when you already know it.
- **Customers.** Name each customer by id; an inline profile goes alongside its id, never instead of it (an inline
  `originator` or `beneficiary` without its `*_id` is rejected). Your customer is registered once
  ([Customers](customers.md)), so `originator_id` is enough. The beneficiary is the counterparty's customer: give
  them an id of your own as `beneficiary_id` and the details the component collected inline as `beneficiary`.
- **Self-hosted destination** (the customer's own wallet): the component's `beneficiary_proof_of_ownership`, such as
  an `eip-191` signature, goes with the body; inline proofs need a DID as `beneficiary_id`. See
  [Wallet addresses](wallets.md).
- **Your own valuation.** If you price the asset yourself, pass `fiat_amount_usd`; it decides the threshold.
- **Your own encryption.** To keep the personal data from Notabene, send it sealed to the receiving institution's
  key as `ivms_encrypted` instead of `ivms` ([Customers](customers.md#where-the-personal-data-is-held)).

## 3. Answer requirements

A `requirements.pending.v3` webhook lists what is outstanding. Your authorization rules, under
**Authorization Rules → Policies**, resolve most of it: screening, limits, whether the counterparty is trusted.
Your code acts only on items routed to it, those whose `agentId` is your DID. Each item's `type` is PascalCase
(`BeneficiaryNameMatch`), while the resolve path takes the kebab-case name (`beneficiary-name-match`); its `id` is the
`evaluationId` that skip takes:

| Requirement | Do this |
|---|---|
| Travel Rule data requested | `POST /api/v1/transfers/{id}/send-presentation` — no body; the data comes from the transfer |
| `beneficiary-name-match`, `internal-compliance-check`, `wallet-address-confirmation` | `POST /api/v1/transfers/{id}/requirements/{requirement}/resolve` with an `outcome` |
| One of yours that does not apply | `POST /api/v1/transfers/{id}/requirements/{evaluationId}/skip` |

Most transfers need nothing here: your rules decide, and Notabene sends your data when it is due.

## 4. Wait for `ReadyToSettle`

`transfer.status_changed.v3` with `toStatus: "ReadyToSettle"` means every institution on the transfer has authorized.
**Broadcast only after this event.** Your own authorization is not an API call: your authorization rules give it
when they clear, and a transfer they flag waits for your compliance team in the dashboard. Call
`POST /api/v1/transfers/{id}/authorize` only for a decision your rules leave to your own system.

`Rejected` or `Canceled` means do not broadcast: return the funds to the customer's balance and show the reason.

## 5. Broadcast, then settle

Broadcast on-chain, then record it:

```bash
curl -s https://sandbox.notabene.id/api/v1/transfers/$ID/settle \
  -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"settlement_id":"0x9fc76417374aa880d4449a1f7f31ec597f00b1f6f3dd2d66f4c9c6c445836d8b"}'
```

Done when the transfer reads `Settled`.

## When things change

| Situation | Call |
|---|---|
| Customer abandons before broadcast | `POST /api/v1/transfers/{id}/cancel` |
| Funds must come back after settlement | `POST /api/v1/transfers/{id}/revert` with a `reason` |
| Already broadcast before creating the transfer | Create with `settlement_id`; the data exchange still runs |
| Wrong receiving institution | `POST /api/v1/transfers/{id}/agents/replace` |
| Matching chain activity to transfers | `GET /api/v1/transfers/match` |
