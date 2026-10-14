# Receive a transfer

Your customer is receiving crypto. Notabene Transact tells you who is sending it and asks you to confirm the address is
yours and that the beneficiary is your customer; you authorize, the sender settles, you credit.

## Quick start

When crypto is coming to one of your customers, the sender tells you who it is from; you say yes or no.

1. Register each deposit address with `POST /api/v1/wallets` when you hand it to a customer ([Wallet
   addresses](wallets.md)).
2. On `transfer.created.v3` with `direction: "Incoming"`, read the transfer and check the beneficiary is your
   customer.
3. `POST /api/v1/transfers/{id}/authorize` with the address the funds land on, or `…/reject`.
4. Credit your customer when the transfer reads `Settled`.

There are two shapes. Work out which one you are in first:

- **Announced before settlement.** The sending institution creates the transfer before it broadcasts. You decide; it
  settles. This is the common case between institutions on Notabene.
- **Seen on-chain first.** Funds arrive with no Travel Rule data attached. You create the record and Notabene finds the
  sender.

## Do this once: register your deposit addresses

Register every deposit address as you issue it, with the customer it belongs to ([Wallet addresses](wallets.md)).
Senders then find you from the address, and Notabene confirms the address is yours without asking you.

## Announced before settlement

1. **Notice it.** `transfer.created.v3` arrives with `direction: "Incoming"`, then `requirements.pending.v3`. Read the
   full transfer with `GET /api/v1/transfers/{id}`.
2. **Answer the requirements whose `agentId` is your DID:**
   - `wallet-address-confirmation` — is the address yours? Resolve with `outcome: "confirmed"` or `"rejected"`. Not
     asked for addresses you registered.
   - `beneficiary-name-match` — does the name match your customer? Resolve with `matched`, `partial_match` (holds the
     transfer for your compliance team) or `mismatch` (final).
   - Travel Rule data owed back → `POST /api/v1/transfers/{id}/send-presentation`.
3. **Decide.** Authorize with the address the funds should land on:

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

## Seen on-chain first

1. **Create the record** from your deposit listener with `POST /api/v1/transfers`. Name yourself as the beneficiary's
   institution (`beneficiary_entity_did`, your DID from `GET /api/v1/identity`) and the receiving customer: that is
   what makes this a deposit rather than a withdrawal.

   ```json
   {
     "ref": "a3c1d9b0-4f7e-4f1e-8a51-0d2b6c9e7f13",
     "asset": "USDC",
     "amount": "250.00",
     "beneficiary_entity_did": "did:web:your-institution.example",
     "beneficiary_id": "urn:uuid:0c8f6a52-6e83-4f2b-9d0a-7b1f0e7c2a41",
     "settlement_id": "0x9fc76417374aa880d4449a1f7f31ec597f00b1f6f3dd2d66f4c9c6c445836d8b",
     "source_address": "0x6b175474e89094c44da98b954eedeac495271d0f",
     "settlement_address": "0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984"
   }
   ```

   The sending institution is discovered from `source_address`. If you know it, also set `originator_entity_did`; a
   `settlement_id` is then required.
2. **Keep answering requirements.** The transfer starts as `SettledIncompleteAuthorization`: the funds moved, decisions
   are outstanding. It is not finished until it reads `Settled`.

Done when the transfer reads `Settled` and your ledger entry carries its `settlement_id`.

## Address ownership requests

`relationship.confirmation_requested.v3` asks you to confirm that an address on a transfer belongs to one of your
customers. Answer with `POST /api/v1/transfers/{id}/requirements/wallet-address-confirmation/resolve` and
`{"outcome":"confirmed"}` or `"rejected"` ([Wallet addresses](wallets.md#when-someone-asks-whether-an-address-is-yours)).
Registering your addresses as you issue them means most of these never reach you.
