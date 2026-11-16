# Dashboard

## Purpose

A small React app to create agreements without Salesforce, watch them move through the pipeline, and download vaulted documents.

## Runs as

A React single-page app (TypeScript, built with Vite), deployed as a static site through SST.

## Screens

| Screen | Contents | Calls |
|---|---|---|
| Agreement list | Customer, amount, status, last updated; newest first | `GET /agreements` |
| Create agreement | Form: template, customer, amount, currency, signer name and email | `POST /agreements` with a generated idempotency key |
| Agreement detail | Current status and a timeline of every event; download buttons once vaulted | `GET /agreements/{id}`, then the Vault Service download route |
| Webhooks | Subscriptions, failed deliveries, a replay button | Notifier routes |

The list and detail screens poll every few seconds while an agreement is in progress. There is no push channel.

## Authentication

A single shared API key, entered once and kept in the browser's local storage. There are no user accounts or roles. This is a deliberate limit of the learning build.

## Observability

The dashboard sends a `traceparent` header with each request, so a trace starts at the button click. See [observability](../observability.md).

## Done when

- Submitting the create form shows the new agreement in the list with status `REQUESTED`.
- The detail screen shows the timeline advancing through each stage without a page reload.
- After vaulting, both download buttons return the files.
- A failed webhook delivery appears on the webhooks screen and can be replayed.
