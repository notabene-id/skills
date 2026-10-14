# Send a transfer

Your customer is withdrawing crypto to an address outside your institution. Notabene Transact works out whether the
Travel Rule applies, finds the beneficiary's institution, exchanges the required data with it, and tells you when it is
safe to broadcast. Your code supplies the transfer and reacts to two events.

## Quick start

Before you send a customer's crypto out, tell Notabene, and wait for its go-ahead.

1. `POST /api/v1/transfers` with the amount, destination address and both customers.
2. Wait for the `transfer.status_changed.v3` webhook with `toStatus: "ReadyToSettle"`.
3. Broadcast on-chain as you do today.
4. `POST /api/v1/transfers/{id}/settle` with the transaction hash.

Most transfers need nothing else. The steps below add the optional pre-check and the cases that need you.

## 1. Assess (optional, recommended)

Before you show the withdrawal confirmation, ask what the transfer will need. Assess creates nothing.

```bash
curl -s https://sandbox.notabene.id/api/v1/transfers/assess \
  -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"asset":"USDC","amount":"250.00","destinationAddress":"0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984"}'
```

- `travelRule` is `Applies`, `Exempt` or `Undetermined`.
- `required` lists the personal-data fields to collect from your customer when it applies.
- `beneficiaryVasp` names the receiving institution when it can be discovered from the address.

Use `required` to decide which fields your withdrawal form asks for. Collect nothing more.

## 2. Create the transfer

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
- **The receiving institution** is discovered from `settlement_address`. Set `beneficiary_entity_did` when you already
  know it.
- **Customers.** Every transfer names both customers by id. Your customer is registered once
  ([Customers](customers.md)), so `originator_id` is enough. The beneficiary is the counterparty's customer: give
  them an id of your own and the details you know inline as `beneficiary`.
- **Self-hosted destination** (the customer's own wallet): attach `beneficiary_proof_of_ownership`, such as an
  `eip-191` signature; inline proofs need a DID as `beneficiary_id`. See [Wallet addresses](wallets.md).
- **Your own valuation.** If you price the asset yourself, pass `fiat_amount_usd`; it decides the threshold.

## 3. Answer requirements

A `requirements.pending.v3` webhook lists what is outstanding. Act on each item whose `agentId` is your DID:

| Requirement | Do this |
|---|---|
| Travel Rule data requested | `POST /api/v1/transfers/{id}/send-presentation` — no body; the data comes from the transfer |
| `beneficiary-name-match`, `internal-compliance-check`, `wallet-address-confirmation` | `POST /api/v1/transfers/{id}/requirements/{requirement}/resolve` with an `outcome` |
| One of yours that does not apply | `POST /api/v1/transfers/{id}/requirements/{evaluationId}/skip` |

Most transfers need nothing here: your policies decide automatically, and Notabene sends your data when it is due.

## 4. Wait for `ReadyToSettle`

`transfer.status_changed.v3` with `toStatus: "ReadyToSettle"` means every institution on the transfer has authorized.
**Broadcast only after this event.** Your own authorization is sent for you when your policies clear; call
`POST /api/v1/transfers/{id}/authorize` only to authorize by hand.

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
