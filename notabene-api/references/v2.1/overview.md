# Overview

How Notabene Transact works and where to start: the model in payment terms, what Notabene handles and what stays
yours, and the path from your first call to live traffic.

## In one minute

When your customer sends crypto to another company, the law asks you to tell that company who is sending and who is
receiving, and to wait for it to agree, before the money moves. A crypto address does not say which company is behind
it, so Notabene works that out, passes the details across, and collects the answers. You keep your wallets and send
the crypto yourself; Notabene tells you when it is safe to.

Three things are enough to start: a **transfer** is one payment; a **customer** is the person on each end; a
**wallet address** is where the crypto lands. The sections below explain each in payment-API terms.

## Where to start

1. Settle your integration type: which way crypto moves, who holds your wallets, and whether your app checks
   withdrawals as customers type ([Go live](go-live.md#what-each-answer-means)). In the dashboard, **Integration → Go
   Live** saves that type and tracks every step it needs, read from what your institution has actually set up and
   sent.
2. [Authentication and API conventions](authentication.md). Create an API credential and swap it for a token.
3. Build the steps in the order [Go live](go-live.md) lists them. Each links to the guide that shows the calls.
4. [Pre-production checklist](pre-production.md). Before real traffic, check what only your code can show.

Build in sandbox first: it has its own account, credentials and data, and nothing there reaches production. When it
all works, set up production the same way. In production, transfers move real funds, so run a transfer end to end in
sandbox before you switch.

## How it works

When your customer withdraws crypto, the Travel Rule requires you to send who they are, and who they are paying, to
the institution receiving the funds: the same originator and beneficiary information a wire transfer carries.
Notabene Transact finds that institution, exchanges the data with it, and tells you when it is safe to send.

```text
Your customer (originator) → your institution (sending) → counterparty institution (receiving) → their customer (beneficiary)
```

1. You create a transfer: the amount, the destination address, and both customers.
2. Notabene identifies the institution behind the address and exchanges the Travel Rule data with it.
3. Each institution authorizes. Webhooks tell you what is still needed, then that the transfer is `ReadyToSettle`.
4. You broadcast on-chain, as you do today, and tell Notabene the transaction hash.

Deposits run the same chain in the other direction: the sender's institution creates the transfer, and you
authorize it. [Send a transfer](transact-send.md) and [Receive a transfer](transact-receive.md) map each step to the
call your code makes.

## The Travel Rule, in payment terms

A wire transfer carries the originator's and the beneficiary's details from bank to bank. The Travel Rule (FATF
Recommendation 16) applies the same requirement to crypto: when your customer sends crypto to another institution,
you must send who they are and who they are paying, and the receiving institution must check it, before the funds
move.

Crypto makes that harder than it is for a wire:

- **An address says nothing about who holds it.** There is no BIC or sort code in `0x1f98…F984`. Notabene works out
  which institution, if any, controls the address.
- **There is no shared messaging network by default.** Notabene is that network: it delivers the data to the
  counterparty institution and brings back its answer.
- **Settlement is final.** Once broadcast, a transaction cannot be recalled, so every check has to clear first.

## What Notabene does, and what stays yours

| Notabene | You |
|---|---|
| Decides whether the Travel Rule applies to a transfer | Create the transfer from your withdrawal or deposit flow |
| Finds the counterparty institution behind an address | Supply your customer's details, and collect what is required |
| Exchanges the Travel Rule data and collects each institution's decision | Authorize or reject what is waiting on you |
| Tracks the transfer's status and tells you by webhook | Broadcast on-chain, and report the transaction hash |

Notabene never holds keys or moves funds. Your wallets, custody and broadcasting stay exactly as they are; Notabene
tells you when it is safe to broadcast.

## The pieces

| Term | What it is | Payment-API equivalent |
|---|---|---|
| **Institution** | A business that holds customers' crypto: you, and the counterparty on the other side. Identified by a DID. | A bank, identified by its BIC |
| **Customer** | The person or business sending (the **originator**) or receiving (the **beneficiary**). Personal data travels as IVMS-101. | The account holder on a wire |
| **Transfer** | One movement of crypto, with its customers, institutions and status. | A payment, or a payment intent |
| **Agent** | An institution as it acts on one transfer. Each agent authorizes or rejects. | A bank in the payment chain |
| **Requirement** | Something the transfer needs before it can settle: data to supply, an address to confirm, a decision to make. | A payment that requires action |
| **Wallet** | A deposit address you registered, linked to the customer it belongs to. | A beneficiary account |

## A transfer's life

1. **Created.** You create it, or a counterparty announces one to you. Each create carries your `ref`, which makes
   retries safe the way an idempotency key does.
2. **Active.** Notabene identifies the counterparty, exchanges the data, and raises requirements. Each is addressed
   to one agent; `requirements.pending.v3` tells you when one is yours.
3. **ReadyToSettle.** Every agent has authorized. For a withdrawal, this is your signal to broadcast. Until then, do
   not move funds.
4. **Settled.** You report the transaction hash, and the transfer is done. A transfer can also end `Rejected` or
   `Canceled`, in which case nothing should move.

The status lives on the transfer, and webhooks tell you when it changes. Treat a webhook as a prompt to re-read the
transfer, as you would with any payment API, rather than as the record itself.

## What you build

| Area | Your side |
|---|---|
| Withdrawals | Create the transfer from your withdrawal flow, answer requirements, broadcast on `ReadyToSettle`, then settle. [Send a transfer](transact-send.md) |
| Deposits | Authorize or reject transfers sent to your customers. [Receive a transfer](transact-receive.md) |
| Customers | Register each customer once with their details, then name them by id on transfers. [Customers](customers.md) |
| Wallet addresses | Register each deposit address you issue, so senders find you and confirmations answer themselves. [Wallet addresses](wallets.md) |

If a wallet provider such as Fireblocks holds your addresses, connecting it removes most of that work. [Go
live](go-live.md) explains each integration type and tracks the steps for yours.
