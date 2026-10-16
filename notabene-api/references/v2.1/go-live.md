# Go live

Every step an integration needs before live traffic, by integration type, as a checklist to apply to the code and
setup. The dashboard's **Integration → Go Live** tracks the same steps: it saves the institution's integration type
and shows, per environment, which steps are done, read from what has actually been set up and sent.

## Your integration type

Three answers decide which steps you need: which way crypto moves, who holds your wallets, and where your customers
enter withdrawal details. [What each answer means](#what-each-answer-means) is at the end of this guide. Infer them from
the code where you can and confirm with the user; if an integration type is already saved under **Integration → Go
Live**, reconcile with it rather than picking a different one silently.

## The steps, in order

Every integration type takes some of these steps, in this order:

1. **Wallet provider** (wallet-managed only). Connect it under **Integration → Wallet Providers**.
2. **API credential.** Create one under **Integration → API Credentials**. [Authentication](authentication.md)
3. **Webhook endpoint.** Add one under **Integration → Webhooks**, then send it a test event. [Webhooks](webhooks.md)
4. **Customers.** Register the customers you transfer for. [Customers](customers.md)
5. **Deposit addresses** (deposits). Register each one as you hand it out. [Wallet addresses](wallets.md)
6. **Customer tokens** (embedded checks). [Embedded checks](transact-embedded.md)
7. **A first withdrawal, then its settlement** (withdrawals). [Send a transfer](transact-send.md)
8. **A first deposit, then its settlement** (deposits). [Receive a transfer](transact-receive.md)

The setup steps (1–3) happen in the dashboard, not in code. The rest are code paths; each is done when the dashboard
shows it in the environment you are working in.

When every step is done, work through the [Pre-production checklist](pre-production.md): what only your code can
show.

Sandbox and production are separate: each has its own credentials, webhook endpoint, customers, addresses and
integration type, and its own progress under **Integration → Go Live**; they share nothing but the login. Do every
step in sandbox first, then again in production, even the ones already done in sandbox.

## Working through it

Build the integration type's steps in the order above, following the guide named for each, then check the code
against the [Pre-production checklist](pre-production.md).

## What each answer means

Not sure? Start with the default: **withdrawals and deposits, self-managed, server-only**. Every other type is that
one with steps removed or added.

| Question | Answer | What changes |
|---|---|---|
| Which way does crypto move? | Withdrawals only | You create transfers when customers send out ([Send a transfer](transact-send.md)); no deposit steps. |
| | Deposits only | You register deposit addresses and answer transfers sent to them ([Receive a transfer](transact-receive.md)); no withdrawal steps. |
| | Both | Both sets of steps. |
| Who holds the wallets? | Self-managed (you, or a custodian not connected to Notabene) | You register customers and deposit addresses, create transfers and handle every webhook. |
| | A connected wallet provider (for example Fireblocks) | Connect it under **Integration → Wallet Providers**; it calls Notabene for the transfers it processes. Register the addresses it holds with the provider as `custodian` ([Wallet addresses](wallets.md)), and handle the events it does not. |
| Where do customers enter withdrawal details? | Server-only | Your backend already has the beneficiary and destination, and calls the API. Nothing to add. |
| | Embedded checks in your app | Your backend mints a short-lived customer token; your front end checks the Travel Rule as customers type ([Embedded checks](transact-embedded.md)). |

The minimal calls and events per direction:

| Direction | Calls | Events to handle |
|---|---|---|
| Withdrawals | assess (optional) → create → settle | `requirements.pending.v3`, `transfer.status_changed.v3` |
| Deposits | register wallets once → authorize or reject → (deposits seen first: create with `settlement_id`) | `transfer.created.v3`, `requirements.pending.v3`, `transfer.status_changed.v3` |
