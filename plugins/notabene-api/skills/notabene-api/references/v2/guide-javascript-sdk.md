# Notabene JavaScript SDK Guide

The Notabene JavaScript SDK (`@notabene/javascript-sdk`) provides embeddable UI components — collectively called **SafeConnect** — that handle Travel Rule data collection in your frontend. They are rendered in an iframe pointing to `https://embedded.notabene.id`; the SDK handles iframe setup, configuration and two-way communication, so you never touch the iframe directly.

## Contents

- [Components at a glance](#components-at-a-glance)
- [Installation](#installation)
- [Authentication: Customer Token](#authentication-customer-token)
- [SDK Initialization](#sdk-initialization)
- [Locale support](#locale-support)
- [Withdrawal Assist](#withdrawal-assist)
- [Deposit Request](#deposit-request)
- [Connect Wallet (Self-Hosted Wallet Verification)](#connect-wallet-self-hosted-wallet-verification)
- [Deposit Assist](#deposit-assist)
- [Component Response Object](#component-response-object)
- [Counterparty Assist (built-in feature)](#counterparty-assist-built-in-feature)
- [Integration checklist](#integration-checklist)

Full docs: [devx.notabene.id/docs/components-overview](https://devx.notabene.id/docs/components-overview)
npm: [@notabene/javascript-sdk](https://www.npmjs.com/package/@notabene/javascript-sdk)
Source: [gitlab.com/notabene/open-source/javascript-sdk](https://gitlab.com/notabene/open-source/javascript-sdk)

---

## Components at a glance

| Component | When to use |
|---|---|
| **Withdrawal Assist** | On your withdrawal screen, before the transfer is created — collects beneficiary info and/or counterparty identification from your customer |
| **Deposit Request** | On your deposit screen — enriches your deposit address/QR code so the sender's VASP can pre-fill transfer information for smoother, compliant deposits |
| **Connect Wallet** | When you need to verify that a customer controls a self-hosted (unhosted) wallet |
| **Deposit Assist** | Post-deposit, when Travel Rule data arrived incomplete — collects missing originator info from your depositing customer |

---

## Installation

`npm install @notabene/javascript-sdk`, or via CDN:
`<script src="https://unpkg.com/@notabene/javascript-sdk/dist/notabene.js"></script>`

---

## Authentication: Customer Token

**Never** use your API access token ([how to get one](guide-travel-rule-api.md#authentication)) in the frontend. The SDK requires a short-lived **customer token** scoped to a specific user, generated server-side.

### Generate on your backend

```
POST /entities/{entityDID}/delegateToken
Authorization: Bearer <your_api_access_token>
Content-Type: application/json
```

```json
{
  "delegateId": "unique-stable-customer-id",
  "entityDID": "did:web:your-vasp.com"
}
```

`entityDID` must match the one in the path.

**Response:** `{ "access_token": "eyJhbGc...", "delegateDid": "did:…", "token_type": "Bearer" }`. Pass
`access_token` to the SDK as `authToken`.

> **Important:** `delegateId` must be unique and stable per customer. Reusing the same id across different customers breaks ownership proof reusability. Generate the token server-side immediately before rendering the component — do not cache it long-term.

Reference: [devx.notabene.id/reference/createdelegatetoken](https://devx.notabene.id/reference/createdelegatetoken)

---

## SDK Initialization

```javascript
import Notabene from '@notabene/javascript-sdk';

const notabene = new Notabene({
  nodeUrl: 'https://api.eu1.notabene.id',  // your region's API host (EU here; US: api.us1)
  authToken: customerToken,                 // access_token from delegateToken — NOT your API access token
});
```

The SDK uses `nodeUrl` to determine which environment the embedded UI connects to (`production` vs `development`). For production use your region's host: `https://api.eu1.notabene.id` (EU) or `https://api.us1.notabene.id` (US).

---

## Locale support

Pass a BCP 47 locale code to render the component in the user's language. The component validates the locale and falls back to `"en"` if unsupported.

```javascript
const withdrawal = notabene.createWithdrawalAssist(data, {
  locale: 'de',  // e.g. 'en', 'de', 'fr', 'es', 'ja', etc.
});
```

---

## Withdrawal Assist

Use this on your withdrawal screen **before** submitting the transfer to the Travel Rule API. The component handles VASP discovery, PII collection, and agent construction — the `txCreate` output is ready to POST directly to the transfer creation endpoint.

```javascript
const withdrawal = notabene.createWithdrawalAssist({
  asset: 'eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48', // CAIP-19 or shorthand e.g. 'USDC'
  destination: '0xBeneficiaryAddress',
  amountDecimal: 100.0,
  assetPrice: 1.00,        // optional — overrides CoinGecko/CoinMarketCap; required for custom/unlisted assets
  customer: {
    name: 'Jane Smith',    // optional, pre-fills originator name
    email: 'jane@example.com',
  },
});
```

**Render options (choose one):**
```javascript
// Option A: Inline iFrame (mount to a DOM element)
withdrawal.mount('#nb-withdrawal');
const { valid, value, txCreate } = await withdrawal.completion();

// Option B: Modal dialog
const { valid, value, txCreate } = await withdrawal.openModal();

// Option C: Popup window
const { valid, value, txCreate } = await withdrawal.popup();
```

**Handling the response:**
```javascript
if (valid) {
  // txCreate is a fully formed transfer payload ready to POST to /entities/{entityDID}/tx
  await yourBackend.createTransfer(txCreate);
} else {
  // Block the withdrawal and prompt the user to retry
}
```

---

## Deposit Request

Use this on your **deposit screen** to generate an enriched deposit address or QR code. When a sender scans the code, their VASP can pre-fill the transfer context — resulting in fewer data gaps and faster compliance processing on your end.

```javascript
const depositRequest = notabene.createDepositRequest({
  asset: 'USDC',
  destination: '0xYourDepositAddress',
  amountDecimal: 100.0,   // optional
  customer: {
    name: 'Jane Smith',
    email: 'jane@example.com',
  },
});

depositRequest.mount('#nb-deposit-request');
```

Working example: [devx.notabene.id/recipes/deposit-request-basic-html-example](https://devx.notabene.id/recipes/deposit-request-basic-html-example)

---

## Connect Wallet (Self-Hosted Wallet Verification)

Use when your compliance policy requires proof that a customer controls a self-hosted (unhosted) wallet before an outgoing transfer to that address.

```javascript
const connectWallet = notabene.createConnectWallet({
  asset: 'ETH',
  address: '0xUnhostedWalletAddress',
});

const { valid, proof } = await connectWallet.openModal();

if (valid) {
  // proof is a signed attestation — attach to the transfer or store for audit
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
  assetPrice: 1.00,        // optional
  customer: {
    name: 'John Smith',
    email: 'john@example.com',
  },
});

// Render the same way as Withdrawal Assist
deposit.mount('#nb-deposit');
const { valid, value } = await deposit.completion();
```

Further reading: [devx.notabene.id/docs/incomplete-how-to-collect-data](https://devx.notabene.id/docs/incomplete-how-to-collect-data)

---

## Component Response Object

All components return the same shape:

| Field | Description |
|---|---|
| `valid` | `true` if the component collected all required data successfully |
| `value` | Raw collected data (IVMS101 fields and counterparty info) |
| `txCreate` | Ready-to-submit transfer payload for `POST /entities/{entityDID}/tx` (Withdrawal Assist only) |
| `ivms101` | Structured IVMS101 object if available |
| `proof` | Self-hosted wallet ownership proof (Connect Wallet only) |

---

## Counterparty Assist (built-in feature)

Both Withdrawal Assist and Deposit Assist have an optional **Counterparty Assist** mode. When enabled, it generates a secure link you can share with the counterparty so *they* provide their own Travel Rule data directly — offloading data collection to the person best positioned to provide it.

```javascript
const withdrawal = notabene.createWithdrawalAssist(data, {
  counterpartyAssist: true,
});
```

---

## Integration checklist

- [ ] Customer token generated server-side per user (`delegateToken`) with a stable `delegateId`
- [ ] SDK initialized with `nodeUrl` pointing to correct environment
- [ ] Component rendered **before** the transfer is submitted to the API
- [ ] `txCreate` from Withdrawal Assist used as the body for `POST /entities/{entityDID}/tx`
- [ ] Deposit Request component added to deposit screen to improve incoming data quality
- [ ] `valid: false` handled gracefully (block action, prompt retry)
- [ ] Locale set to match your user's language where applicable
