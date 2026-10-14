# Choose your integration

Notabene Transact handles Travel Rule compliance for the crypto your customers send and receive. Two questions decide
the shortest path; answer them, then follow only the guides they name.

## Quick start

Not sure? Build the default: **self-managed, server-only**. Your backend creates transfers for withdrawals and answers
transfers for deposits. Everything else is that path with steps removed (a connected wallet provider) or added (live
checks in your front end).

## 1. Who holds the wallets?

| Your setup | Integration mode | What you build |
|---|---|---|
| A wallet provider (for example Fireblocks) holds your addresses, and you connect it under **Integration → Wallet Providers** in the dashboard | **Wallet-managed** | Connect the provider; it calls Notabene for the transfers it processes. Register the addresses it holds with the provider as `custodian` ([Wallet addresses](wallets.md)), and handle the events your provider does not. |
| You run your own wallets, or your custodian is not connected | **Self-managed** (default) | Register your customers and deposit addresses, create transfers, and handle every webhook. |

Self-managed is the complete path; wallet-managed is the same path with steps removed. If you are unsure, build
self-managed against sandbox: everything in it still works once you connect a provider.

## 2. Where do customers enter withdrawal details?

| Your front end | Pattern |
|---|---|
| Your backend already has everything (the beneficiary's name, the destination) | **Server-only.** Your backend calls the API. Nothing else to add. |
| Customers type the destination in your web or mobile app, and you want to check the Travel Rule requirements as they type | **Embedded.** Your backend mints a short-lived customer token; your front end uses it for lookups and pre-checks. See [Embedded checks with customer tokens](transact-embedded.md). |

## The minimal integration, per direction

| Direction | Calls | Events to handle |
|---|---|---|
| Withdrawals — [Send a transfer](transact-send.md) | assess (optional) → create → settle | `requirements.pending.v3`, `transfer.status_changed.v3` |
| Deposits — [Receive a transfer](transact-receive.md) | register wallets once → authorize or reject → (deposits seen first: create with `settlement_id`) | `transfer.created.v3`, `requirements.pending.v3`, `transfer.status_changed.v3` |

Every path also needs [Getting started](getting-started.md) (credential and token), [Customers](customers.md),
[Wallet addresses](wallets.md) and [Webhooks](webhooks.md). [Go live](go-live.md) tracks each step as you complete
it.
