# Notabene JavaScript SDK Guide

The Notabene JavaScript SDK (`@notabene/javascript-sdk`) provides embeddable UI components — collectively called **SafeConnect** — that handle Travel Rule data collection directly in your frontend. The components are served by a SvelteKit application hosted at `https://connect.notabene.id` (the SDK's default `uxUrl`) and rendered in an iframe, modal or popup that the SDK manages. You configure and communicate with them through the SDK's JavaScript API.

Full docs: [devx.notabene.id/docs/components-overview](https://devx.notabene.id/docs/components-overview)
npm: [@notabene/javascript-sdk](https://www.npmjs.com/package/@notabene/javascript-sdk)
Source: [gitlab.com/notabene/open-source/javascript-sdk](https://gitlab.com/notabene/open-source/javascript-sdk)

---

## Components at a glance

| Component | Factory | When to use |
|---|---|---|
| **Withdrawal Assist** | `createWithdrawalAssist` | On your withdrawal screen, before the transfer is created — collects beneficiary info and/or counterparty identification from your customer |
| **Deposit Request** | `createDepositRequest` | On your deposit screen — enriches your deposit address/QR code so the sender's VASP can pre-fill transfer information for smoother, compliant deposits |
| **Connect Wallet** | `createConnectWallet` | When you need to verify that a customer controls a self-hosted (unhosted) wallet |
| **Deposit Assist** | `createDepositAssist` | Post-deposit, when Travel Rule data arrived incomplete — collects missing originator info from your depositing customer |

> The SDK handles all iframe setup, configuration passing, and two-way communication. You never interact with the iframe directly.

---

## Installation

```bash
npm install @notabene/javascript-sdk
```

Or via CDN (plain HTML):
```html
<script src="https://unpkg.com/@notabene/javascript-sdk/dist/notabene.js"></script>
```

---

## Authentication: Customer (delegate) Token

**Never** use your API access token in the frontend. The SDK requires a short-lived token scoped to one customer, generated server-side. In v2 this is the **delegate token**:

```
POST https://api.eu1.notabene.id/entities/{entityDID}/delegateToken
Authorization: Bearer <your_api_access_token>
Content-Type: application/json
```

```json
{
  "entityDID": "did:web:your-vasp.com",
  "delegateId": "urn:uuid:7d1c2f6e-0b0e-4a8e-9a49-3f1a0c7e2b11"
}
```

`entityDID` must equal the path's `entityDID`. `delegateId` (1–255 chars) is your stable id for the customer; use an IRI (e.g. `urn:uuid:…`), which some deployments enforce.

**Response** `200`:
```json
{
  "access_token": "eyJhbGc...",
  "token_type": "Bearer",
  "delegateDid": "did:key:z6Mk..."
}
```

Pass `access_token` to the SDK as `authToken`. The JWT is valid for **1 hour**; its `sub` is `delegateDid`.

> **Important:** `delegateId` must be unique and stable per customer: the same `delegateId` always maps to the same `delegateDid`, which is what makes ownership proofs reusable. Reusing one id across customers breaks that. Generate the token right before rendering the component — do not cache it.

---

## SDK Initialization

```javascript
import Notabene from '@notabene/javascript-sdk';

const notabene = new Notabene({
  nodeUrl: 'https://api.eu1.notabene.id',  // the API host you call
  authToken: customerToken,                 // delegate token from your backend — NOT your API access token
  locale: 'de',                             // optional, default 'en'
  // uxUrl: 'https://connect.notabene.id',  // optional, this is the default
  // theme: { ... },                        // optional
});
```

Use the same `nodeUrl` you use for the Notabene API. The embedded UI only accepts allow-listed API hosts.

The locale is set once on the constructor (not per component); unsupported locales fall back to `en`.

---

## Withdrawal Assist

Use this on your withdrawal screen **before** submitting the transfer to the Travel Rule API. The component handles VASP discovery, PII collection, and ownership proofs.

```javascript
const withdrawal = notabene.createWithdrawalAssist({
  asset: 'eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48', // CAIP-19 or shorthand e.g. 'USDC'
  destination: '0xBeneficiaryAddress',
  amountDecimal: 100.0,
  assetPrice: { price: 1.0, currency: 'USD' }, // optional; needed for custom/unlisted assets
  customer: {
    name: 'Jane Smith',    // optional, pre-fills originator name
    email: 'jane@example.com',
  },
});
```

Signature: `createWithdrawalAssist(value, options?, callbacks?)`. `options` are `TransactionOptions` (e.g. `proofs`, `counterpartyAssist`, `fields`, `vasps`); `callbacks` is `{ callback?, redirectUri? }` for server callbacks / redirects on completion.

**Render options (choose one):**
```javascript
// Option A: Inline iFrame (mount to a DOM element, CSS selector)
withdrawal.mount('#nb-withdrawal');
const response = await withdrawal.completion();

// Option B: Modal dialog
const response = await withdrawal.openModal();

// Option C: Popup window (needed by many self-hosted wallets that refuse iframes)
const response = await withdrawal.popup();
```

The promise rejects if the user cancels or the component errors. Event style also works: `withdrawal.on('complete' | 'error' | 'warning' | 'ready', cb)` returns an unsubscribe function, and `withdrawal.update(value, options?)` pushes new input.

**Handling the response:**

`response.txCreate` is a **v1** `tx/create` payload (`transactionAsset`, `originatorVASPdid`, …), not a v2 body. For v2, convert the response with the SDK's transformer:

```javascript
import { componentResponseToTxRequests } from '@notabene/javascript-sdk';

if (response.valid) {
  const { createTx, ivms101, confirmRelationship } = componentResponseToTxRequests(
    response,
    customerToken,                          // the delegate token; its sub becomes originator @id
    { referenceId: myPersistedRef, originator: customerIvms }, // your KYC IVMS101 for the customer
  );
  // createTx       -> POST /entities/{entityDID}/tx
  // ivms101        -> POST /entities/{entityDID}/tx/{id}/append
  // confirmRelationship (if a proof was collected) -> PATCH /entities/{entityDID}/relationships
} else {
  // Block the withdrawal and prompt the user to retry
}
```

Pass your own `referenceId` (`ref`); otherwise the transformer uses the response's `transactionId` or a random string, which breaks idempotent retries. `componentResponseToTxCreateRequest` and `componentResponseToIVMS101` produce the two bodies individually.

---

## Deposit Request

Use this on your **deposit screen** to generate an enriched deposit address or QR code. When a sender scans the code, their VASP can pre-fill the transfer context — resulting in fewer data gaps and faster compliance processing on your end.

```javascript
const depositRequest = notabene.createDepositRequest({
  asset: 'USDC',
  destination: '0xYourDepositAddress',  // address or CAIP-10
  amountDecimal: 100.0,
  customer: {
    name: 'Jane Smith',
  },
}, { showQrCode: true });               // options; showQrCode defaults to true

depositRequest.mount('#nb-deposit-request');
```

Working example: [devx.notabene.id/recipes/deposit-request-basic-html-example](https://devx.notabene.id/recipes/deposit-request-basic-html-example)

---

## Connect Wallet (Self-Hosted Wallet Verification)

Use when your compliance policy requires proof that a customer controls a self-hosted (unhosted) wallet before an outgoing transfer to that address.

```javascript
const connectWallet = notabene.createConnectWallet({
  asset: 'ETH',
  address: '0xUnhostedWalletAddress',  // optional; address or CAIP-10. Omit to let the user pick
});

const { valid, proof } = await connectWallet.openModal();

if (valid) {
  // proof.address is the proved account (CAIP-10). Store it with the relationship
  // (PATCH /entities/{entityDID}/relationships with the proof) or keep it for audit.
}
```

Full self-hosted wallet docs: [devx.notabene.id/docs/self-hosted-wallet-verification-1](https://devx.notabene.id/docs/self-hosted-wallet-verification-1)

---

## Deposit Assist

Use this **after** an incoming on-chain deposit where Travel Rule data arrived incomplete. Trigger it from your compliance workflow when a transfer is flagged with missing originator PII.

```javascript
const deposit = notabene.createDepositAssist({
  asset: 'USDC',
  source: '0xOriginatorAddress',
  amountDecimal: 100.0,
  transactionId: transferId,   // optional: the Notabene transfer id, echoed back in the response
  customer: {
    name: 'John Smith',
    email: 'john@example.com',
  },
});

// Render the same way as Withdrawal Assist
deposit.mount('#nb-deposit');
const response = await deposit.completion();
```

Deposits return `txUpdate` (not `txCreate`). `componentResponseToTxRequests(response, customerToken, { beneficiary: customerIvms })` handles both: for deposits the token's `sub` becomes the beneficiary `@id`.

Further reading: [devx.notabene.id/docs/incomplete-how-to-collect-data](https://devx.notabene.id/docs/incomplete-how-to-collect-data)

---

## Component Response Object

Transaction components resolve with a `TransactionResponse`:

| Field | Description |
|---|---|
| `valid` | `true` if the component collected all required data successfully |
| `status` | `empty`, `verify`, `pending`, `verified` or `banned` |
| `errors` | `[{ attribute, message }]` validation errors |
| `requestID` | Echoes the request's `requestId` |
| `value` | The completed input (withdrawal / deposit data incl. counterparty) |
| `ivms101` | IVMS101 object for the counterparty data collected |
| `proof` | Self-hosted wallet ownership proof, when one was collected |
| `txCreate` | v1 `tx/create` payload (withdrawals). Convert with the transformer for v2 |
| `txUpdate` | v1 payload for deposits. Convert with the transformer for v2 |

---

## Counterparty Assist (built-in feature)

Both Withdrawal Assist and Deposit Assist have an optional **Counterparty Assist** mode, off by default. When enabled, it generates a secure link you can share with the counterparty so *they* provide their own Travel Rule data directly (or, for `self`, so the customer can sign a wallet proof on another device).

```javascript
import { PersonType } from '@notabene/javascript-sdk';

const withdrawal = notabene.createWithdrawalAssist(data, {
  counterpartyAssist: { counterpartyTypes: [PersonType.NATURAL, PersonType.LEGAL, PersonType.SELF] },
  // or `true` / `false`
});
```

For third-party counterparties the response carries a `refreshSource` (`{ url, key, id }`) to fetch the encrypted data once the counterparty completes.

---

## Architecture note (for contributors / SDK maintainers)

The embedded UI is a SvelteKit application deployed to Cloudflare Pages:

| Environment | URL | DID |
|---|---|---|
| Production | `https://connect.notabene.id` | `did:web:flow.link` |
| Preview | `https://connect.notabene.dev` | `did:web:dev.flow.link` |

The DID documents are served from `flow.link` / `dev.flow.link` at `/.well-known/did.json` (not from the `connect.*` hosts) and are used for DIDComm messaging. Configuration and the token are passed to each component in the URL (fragment and query). The SDK wraps the component and handles the two-way communication protocol described in `docs/host-spec.md` in the embedded-ui repo.

Components are developed and tested using Storybook (`npm run storybook` → `http://localhost:6006`).

---

## Integration checklist

- [ ] Delegate token generated server-side per customer with a stable `delegateId`, right before rendering
- [ ] SDK initialized with `nodeUrl` set to the API host you call
- [ ] Component rendered **before** the transfer is submitted to the API
- [ ] Component response converted with `componentResponseToTxRequests` (not `txCreate` posted as-is) and a persisted `referenceId`
- [ ] PII appended and any wallet `proof` stored on the relationship
- [ ] Deposit Request component added to deposit screen to improve incoming data quality
- [ ] `valid: false` and cancellation (rejected promise) handled gracefully (block action, prompt retry)
- [ ] `locale` set on the constructor to match your user's language where applicable
