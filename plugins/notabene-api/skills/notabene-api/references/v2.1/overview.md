# Getting Started

Choose how you integrate with Notabene Transact, watch every step your institution needs go green as you build it,
and see in one picture what Notabene does and what stays yours.

## Contents

- [Your integration type](#your-integration-type)
- [Your progress](#your-progress)
- [How it works](#how-it-works)
- [The pieces](#the-pieces)
- [What Notabene does, and what stays yours](#what-notabene-does-and-what-stays-yours)
- [Your DID document](#your-did-document)
- [Before real traffic](#before-real-traffic)

## Your integration type

Three answers decide which steps you need: which way crypto moves, who holds your wallets, and where your customers
enter transfer details. Not sure? Start with the default, **withdrawals and deposits, self-managed, embedded
components**: every other type is that one with steps removed or added. The embedded components are the
recommended way to collect Travel Rule data: Withdrawal Assist in your withdrawal form, Deposit Assist with
Counterparty Assist in your deposit screen. Server-only is for integrations with no web front end.

The dashboard's **Integration → Getting Started** saves the answers and ticks off the steps below as they are done, read from what your institution has actually set up and sent. If an integration type is already saved there, reconcile with it rather than picking a different one silently.

## Your progress

Every integration type takes some of these steps, in this order:

1. **DID document.** Serve the document Notabene issued for your `did:web` on your domain, so counterparties can
   verify you. Notabene checks it is reachable; **My Organization** shows the result. [Below](#your-did-document)
2. **Wallet provider** (wallet-managed only). Connect it under **Integration → Wallet Providers**.
3. **API credential.** Create one under **Integration → API Credentials**. [Authentication](authentication.md)
4. **Webhook endpoint.** Add one under **Integration → Webhooks**, then send it a test event. [Webhooks](webhooks.md)
5. **Blockchain analytics** (optional). Connect a provider under **Integration → Data Providers**; Notabene screens
   every address on a transfer with it as the transfer is created, and your authorization rules read the risk score.
6. **Sanctions screening** (optional). Connect a provider under **Integration → Data Providers**; Notabene screens
   each customer's name with it when their details arrive, and your rules read the match.
7. **Customers.** Register the customers you transfer for. [Customers](customers.md)
8. **Deposit addresses** (deposits). Register each one as you hand it out. [Wallet addresses](wallets.md)
9. **Embedded components** (embedded front end): customer tokens, then Withdrawal Assist for withdrawals and
   Deposit Assist for deposits. [Embedded components](transact-embedded.md)
10. **A first withdrawal, then its settlement** (withdrawals). [Withdrawals](transact-send.md)
11. **A first deposit, then its settlement** (deposits). [Deposits](transact-receive.md)

Sandbox and production are separate: each has its own credentials, endpoint, customers, addresses and integration
type, and its own progress. When every step is done in sandbox, do every step again in production, even the ones already
done in sandbox: the two environments share nothing but your login.

## How it works

When your customer withdraws crypto, the Travel Rule requires you to send who they are, and who they are paying, to
the institution receiving the funds: the same originator and beneficiary information a wire transfer carries.
Notabene Transact finds that institution, exchanges the data with it, and tells you when it is safe to send. Each
step lines up with one call your code makes, one event it handles, or a decision your authorization rules make for
you.

Deposits run the same chain in the other direction: the sender's institution creates the transfer, and your rules
authorize it. [Withdrawals](transact-send.md) and [Deposits](transact-receive.md) map each step to the call your
code makes. Webhooks tell you when the status changes; treat one as a prompt to re-read the transfer, as
you would with any payment API, rather than as the record itself.

## The pieces

The diagram above, piece by piece.

| Term | What it is | Payment-API equivalent |
|---|---|---|
| **Institution** | A business that holds customers' crypto: you, and the counterparty on the other side. Identified by a DID. | A bank, identified by its BIC |
| **Customer** | The person or business sending (the **originator**) or receiving (the **beneficiary**). Personal data travels as IVMS-101. | The account holder on a wire |
| **Transfer** | One movement of crypto, with its customers, institutions and status. | A payment, or a payment intent |
| **Agent** | An institution as it acts on one transfer. Each agent authorizes or rejects. | A bank in the payment chain |
| **Requirement** | Something the transfer needs before it can settle: data to supply, an address to confirm, a decision to make. | A payment that requires action |
| **Wallet** | A deposit address you registered, linked to the customer it belongs to. | A beneficiary account |

## What Notabene does, and what stays yours

| Notabene | You |
|---|---|
| Decides whether the Travel Rule applies to a transfer | Create the transfer from your withdrawal or deposit flow |
| Finds the counterparty institution behind an address | Register your customers, and collect the counterparty's details with the embedded components |
| Screens addresses and names with the providers you connect | Connect your blockchain analytics and sanctions screening providers, if you use them |
| Exchanges the Travel Rule data and collects each institution's decision | Set your authorization rules; they authorize and reject for you, and flag what needs a person |
| Tracks the transfer's status and tells you by webhook | Broadcast on-chain, and report the transaction hash |

Notabene never holds keys or moves funds. Your wallets, custody and broadcasting stay exactly as they are; Notabene
tells you when it is safe to broadcast.

## Your DID document

Notabene created your institution's identity when you registered: a `did:web` on your domain, and the DID document
that carries its keys and the Notabene endpoint counterparties message. You do not create either. What is yours is
to publish the document where a `did:web` is looked up, so every counterparty, on Notabene or on another Travel
Rule network, can verify you.

1. Fetch the document Notabene serves for your DID: `https://sandbox.notabene.id/did/<your DID>/did.json`, with
   your DID from **My Organization** or `GET /api/v1/identity`. No login is needed.
2. Serve exactly those bytes at `https://<your domain>/.well-known/did.json` (or, for a DID with a path such as
   `did:web:acme.example:sandbox`, at `https://acme.example/sandbox/did.json`), as `application/did+json` or
   `application/json`, over HTTPS with a valid certificate.
3. Keep it current: when Notabene rotates a key or an endpoint, fetch and serve it again. Many institutions have
   their site fetch the Notabene URL on each request instead of copying it.

Notabene's network probes the URL and shows the result under **My Organization**, as the step above. Until it is
reachable, counterparties on other networks cannot resolve you, and ones on Notabene fall back to Notabene's copy.
An institution that keeps its own personal-data key adds it to this document as well
([Customers](customers.md#where-the-personal-data-is-held)).

## Before real traffic

The steps above are read from your account. These are what only your code can show: check each before live traffic,
or hand them to a coding agent.

| Check | Why it matters | Guide |
|---|---|---|
| Every transfer create sends a `ref` you stored first, and a retry replays the identical body. | A timeout can hide a transfer that was created; the same `ref` returns it instead of a second one. | [Create the transfer](transact-send.md#2-create-the-transfer) |
| Withdrawals broadcast only after `ReadyToSettle`, then call settle with the transaction hash. | A broadcast cannot be recalled; every institution must have authorized first. | [Broadcast, then settle](transact-send.md#5-broadcast-then-settle) |
| Deposits credit your customer on `Settled`; `SettledIncompleteAuthorization` stays unfinished. | Funds that arrived before the checks cleared still need your decision. | [Not announced](transact-receive.md#without-a-travel-rule-message) |
| Webhook signatures are verified on the raw body, events de-duplicated on `svix-id`, and `2xx` answered within seconds. | Delivery is at least once and in any order, and an unverified event is not from Notabene. | [Handle events safely](webhooks.md#3-handle-events-safely) |
| A daily job lists transfers with `requires_action: true`. | Nothing waits on you because of an event that never arrived. | [Handle events safely](webhooks.md#3-handle-events-safely) |
| Production has its own API credential, webhook secret and token `audience`, kept in your secrets manager. | Each environment is a separate trust boundary, and a sandbox audience answers `401` in production. | [Your base URL](authentication.md#1-your-base-url) |
