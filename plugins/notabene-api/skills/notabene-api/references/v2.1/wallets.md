# Wallet addresses

A crypto address does not say who holds it. Registering the deposit addresses you hand to customers tells Notabene
they are yours, so institutions sending to them can find you, and the question "is this address yours?" is answered
without you.

## Quick start

Tell Notabene about each deposit address when you create it, and which customer it belongs to.

1. When you issue a deposit address, `POST /api/v1/wallets` with the address in CAIP-10 form and the customer's id.
2. For a shared (omnibus) address, leave the customer out.
3. When you retire an address, `DELETE /api/v1/wallets/{address}`.

## Why it matters

- **Senders find you.** A registered address is published to Notabene's address registry as yours, so an
  institution sending to it learns who to exchange Travel Rule data with, without asking you first.
- **Confirmations answer themselves.** Before personal data is shared, the sending side asks whether the address
  belongs to the beneficiary's institution. For a registered address, Notabene answers for you. For an unregistered
  one the question reaches you as a webhook, and the transfer waits on your answer.
- **It fixes the past too.** Registering an address with its customer answers any confirmation already waiting on
  live transfers to it.

## Decisions

| Decision | Options | Recommended |
|---|---|---|
| One address per customer, or shared | `customer_id` set, or omitted for an omnibus address | Per customer where your custody allows; it lets confirmations name the customer. |
| Who holds the keys | You (the default), or a custodian you name with `custodian` | Name the custodian's DID when a wallet provider holds the address for you. |
| Your customer's own wallet | Record it with `managed: false` | Do this only to keep a record: it is never published. Customers' own wallets are proven, not registered (below). |

## Register an address

```bash
curl -s https://sandbox.notabene.id/api/v1/wallets \
  -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"address":"eip155:1:0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984",
       "customer_id":"urn:uuid:0c8f6a52-6e83-4f2b-9d0a-7b1f0e7c2a41"}'
```

- **CAIP-10 only.** The chain prefix is required here (`eip155:1:0x…`), unlike on transfers, which also accept a bare
  address.
- **The customer must be registered first** ([Customers](customers.md)).
- **Registering again** with the same content succeeds and returns the same address; different content for the same
  address answers `409`.
- **Tagged addresses** (a memo or destination tag) register as their own address. Name a custodian on the untagged
  address.

## When someone asks whether an address is yours

`relationship.confirmation_requested.v3` means a counterparty wants to know whether an address on a transfer belongs
to your customer, and it is not registered. Answer it on the transfer:

```bash
curl -s https://sandbox.notabene.id/api/v1/transfers/$ID/requirements/wallet-address-confirmation/resolve \
  -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"outcome":"confirmed"}'
```

`rejected` is final for this transfer: it rejects the transfer and tells the counterparty the address is not yours.
What happens next time depends on who said it was yours:

- **Only a counterparty claimed it.** The rejection also withdraws that claim, so later transfers to the address are
  rejected the same way until you register it.
- **You said so yourself** (you registered it, confirmed it before, or used it on a transfer you created). The
  rejection answers this transfer only; a later transfer to the address can still be confirmed. De-register the
  wallet (`DELETE /api/v1/wallets/{address}`) to stop that.

Registering your addresses as you issue them means this rarely reaches you.

## Your customer's own wallet

When a customer withdraws to a wallet they control themselves (a self-hosted wallet), there is no institution on the
other side to confirm it. Instead the customer proves they own it:

1. Your front end asks the customer to sign a message with that wallet, for example an `eip-191` signature on
   Ethereum.
2. Your backend submits it: inline on the transfer as `beneficiary_proof_of_ownership`, or on its own with
   `POST /api/v1/customers/{customer}/wallets/{address}/proofs`.
3. A valid signature is `Verified` straight away. A `screenshot` or `self-declaration` stays `Submitted` until your
   team accepts or rejects it (`POST /api/v1/wallet-proofs/{id}/accept`, `…/reject`).

Signatures are verified for Ethereum and EVM chains (`eip-191`), Bitcoin (`bip-322`, `bip-137`), Solana
(`ed25519`), Stellar, Tron, XRP and Cardano. A proof inline on the transfer needs the customer's id to be a DID; the
standalone call takes any id. `GET` on the same path lists the proofs already submitted.

## Who holds an address?

`POST /api/v1/wallet-ownerships/discover` with an address tells you which institution holds it, if Notabene knows. It
changes nothing, so it is safe to call while a customer types a destination.

## With a wallet provider

If a wallet provider such as Fireblocks holds your addresses, connect it under **Integration → Wallet Providers**.
It then calls Notabene for the transfers it processes. Register the addresses it holds with `custodian` set to the
provider, so senders and confirmations resolve to you.
