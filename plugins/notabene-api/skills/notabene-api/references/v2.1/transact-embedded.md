# Collect customer data with embedded components

Travel Rule data comes from your customer: who they are sending to, which institution or self-hosted wallet holds the
destination, and proof that they own that wallet. Notabene's JavaScript SDK puts components into your web app that
collect all of it, in your customer's language, and hand your backend the transfer to post. They are the
recommended way to collect it: [Withdrawals](transact-send.md) mounts Withdrawal Assist and
[Deposits](transact-receive.md) mounts Deposit Assist with Counterparty Assist; this page is the full
reference. The only API call this step adds is minting a customer token; your credential never leaves your backend.

## Contents

- [Quick start](#quick-start)
- [1. Mint a customer token (backend)](#1-mint-a-customer-token-backend)
- [2. Mount a component (front end)](#2-mount-a-component-front-end)
- [3. Post what it collected (backend)](#3-post-what-it-collected-backend)
- [Counterparty Assist (optional)](#counterparty-assist-optional)
- [Revoking](#revoking)

## Quick start

Your backend mints a short-lived token for one customer, the components work for that customer with it, and your
backend posts what they collect.

1. Backend: `POST /api/v1/entities/{your DID}/delegate-token` with the customer's id, behind an endpoint only your
   signed-in customer can reach.
2. Front end: `npm install @notabene/javascript-sdk`, create `Notabene` from `@notabene/javascript-sdk/src/v21`
   with a `fetchToken` that calls that endpoint, and mount `createWithdrawalAssist` in your withdrawal form.
3. Backend: take the `transfer` the component completes with, add `ref`, and `POST /api/v1/transfers` with your
   own credential, as in [Withdrawals](transact-send.md).

## 1. Mint a customer token (backend)

The customer must already exist ([Customers](customers.md)); use their `id` as `customer_ref`. Minting for an unknown
customer returns `400`.

```bash
curl -s https://sandbox.notabene.id/api/v1/entities/$YOUR_DID/delegate-token \
  -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"customer_ref":"urn:uuid:0c8f6a52-6e83-4f2b-9d0a-7b1f0e7c2a41"}'
```

`$YOUR_DID` is your institution's DID (from `GET /api/v1/identity`). The response carries `access_token`,
`expires_in` (about an hour) and `jti`. Put the call behind an endpoint that mints for the customer who is signed
in, never for one the browser names. The token is bound to that customer and read-only: it lets the components work
for them and nothing else. It cannot create a transfer, and the routes it opens are for the components, not for
your code.

## 2. Mount a component (front end)

Install `@notabene/javascript-sdk` (2.24 or later) and create the v2.1 `Notabene` with a `fetchToken` that asks your
backend for a token. It is called once at start and again shortly before the token expires. `nodeUrl` is the base URL
your backend calls.

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
```

Three components. Each mounts in your page (`mount('#id')`), opens as a modal (`openModal()`) or a popup window
(`popup()`, for wallets that refuse to run in a frame), and resolves with what it collected.

| Component | Use it when | Completes with |
|---|---|---|
| `createWithdrawalAssist({ asset, amountDecimal, destination })` | A customer withdraws: it finds who holds the destination, asks for the beneficiary's details, and collects a proof when the wallet is self-hosted. | `{ transfer }` |
| `createDepositAssist({ asset, amountDecimal, source })` | A deposit nobody announced, arrived or expected: it asks your customer for the originator's details, or lets them share a link so the sender enters their own. | `{ transfer }` |
| `createConnectWallet({ asset, address })` | A customer proves they own a self-hosted wallet, on its own. | `{ wallet }` |

```js
const withdrawal = notabene.createWithdrawalAssist({
  asset: 'ETH',
  amountDecimal: 0.25,
  destination: '0x8d12a197cb00d4747a1fe03395095ce2a5cc6819',
});
withdrawal.mount('#nb-withdrawal');
// As the customer edits the form:
withdrawal.update({ amountDecimal: 0.5 });

const { transfer } = await withdrawal.completion();
await fetch('/api/withdrawals', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ transfer }),
});
```

Every field is optional: the component asks the customer for whatever you leave out, and `update()` feeds it what
they type. A second argument configures it: which counterparty types to allow (`self`, `natural`, `legal`), which
IVMS-101 fields to ask for, which ownership proofs to accept, `jurisdiction`, and `hide` for sections your form
already covers. The SDK's README lists every option.

## 3. Post what it collected (backend)

`transfer` is the `POST /api/v1/transfers` body the component filled in: `asset`, `amount`, the customers' ids or
IVMS-101 profiles, `settlement_address` (or `source_address` for a deposit), the counterparty institution's DID when
it found one, and the ownership proof when the wallet is self-hosted. Your backend adds `ref` (a stored UUID), sets
your customer's id if the component left it out (`originator_id` on a withdrawal, `beneficiary_id` on a deposit),
adds `settlement_id` (the transaction hash) for a deposit, and posts it with your credential. From there it is a
transfer like any other: [Withdrawals](transact-send.md) or [Deposits](transact-receive.md).

`wallet` from Connect Wallet is `{ address, proof }`: file the proof with
`POST /api/v1/customers/{customer}/wallets/{address}/proofs` ([Wallet addresses](wallets.md)), or keep it to send as
`beneficiary_proof_of_ownership` on the next transfer to that address.

## Counterparty Assist (optional)

The components can let your customer share a link, so the counterparty enters their own details or the customer
finishes a wallet proof on another device. To offer it, give `Notabene` a `fetchCounterpartyAssistToken` that calls a
second backend endpoint, which mints with `POST /api/v1/entities/{your DID}/counterparty-assist-token` for the same
`customer_ref` and returns `{ token: access_token, jti }`; then set `counterpartyAssist: { counterpartyTypes: [...] }`
in the component's options. When the customer shares a link, the component completes with `{ refreshSource }`
instead of a transfer; `getRefreshResult(refreshSource)` from the same module reads the counterparty's `{ transfer }`
once they are done. Revoke that link's token afterwards with `POST …/delegate-token/{jti}/revoke`.

## Revoking

`POST /api/v1/entities/{your DID}/delegate-token/revoke-all` with the `customer_ref` invalidates every outstanding
token for that customer, for example when they sign out everywhere or their account is closed.
`POST …/delegate-token/{jti}/revoke` invalidates one.
