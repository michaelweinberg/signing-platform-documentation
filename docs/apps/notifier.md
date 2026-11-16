# Notifier

## Purpose

The only service that tells the outside world what happened. It writes agreement status back to Salesforce and delivers outbound webhooks to registered subscribers.

## Runs as

- A long-running NestJS Kafka microservice (`@nestjs/microservices` with the Kafka transport) in a container: Docker locally, one small Fargate task when deployed, with an OpenTelemetry collector sidecar.
- Lambda HTTP handlers for subscription management and replay. These share the Notifier's package and table, and avoid the cost of a load balancer in front of the Fargate task.

This is the one consumer that manages its own Kafka connection, consumer group, and offsets. Every other consumer is a Lambda event source mapping.

## Events

| Direction | Topic | Event |
|---|---|---|
| Consumes | `agreements`, `documents`, `envelopes`, `vault` | all |
| Produces | none | |

## Behaviour

### Salesforce write-back

For each event that carries a status change (see the status table in the [event catalogue](../event-catalogue.md)), update the Opportunity:

| Opportunity field | Value |
|---|---|
| `Agreement_Status__c` | The agreement status |
| `Vault_Reference__c` | The vault record ID, set on `DocumentVaulted` |

The Opportunity ID comes from `AgreementRequested`, so the Notifier stores a small agreement-to-Opportunity mapping when it sees that event. Agreements created from the dashboard have no Opportunity and are skipped.

### Outbound webhooks

For each event, the Notifier creates one delivery record per active subscription whose event filter matches, then attempts delivery.

Each delivery is an HTTP `POST` of the event as JSON, with these headers:

| Header | Value |
|---|---|
| `X-Signature` | HMAC-SHA256 of `<timestamp>.<body>` using the subscription's secret |
| `X-Timestamp` | Unix time of the attempt; lets the receiver reject old replays |
| `X-Event-Id` | The event ID; lets the receiver deduplicate |

A `2xx` response within 10 seconds is success. Anything else schedules a retry with exponential backoff (1 minute, doubling, with jitter) up to 8 attempts, after which the delivery is marked `FAILED`.

A scheduler inside the process queries for due deliveries every 30 seconds. Because delivery state is in DynamoDB, a restart of the task loses nothing.

### HTTP routes

All require the shared API key.

| Route | Behaviour |
|---|---|
| `POST /webhook-subscriptions` | Registers a URL and an event filter; returns the signing secret once |
| `GET /webhook-subscriptions` | Lists subscriptions and recent delivery counts |
| `POST /webhook-deliveries/{id}/replay` | Resets a `FAILED` delivery so the scheduler attempts it again |

## Data

**Notifier table**

| Item | Key | Contents |
|---|---|---|
| Subscription | `PK=SUBSCRIPTION#<id>` | URL, event filter, secret, active flag |
| Delivery | `PK=DELIVERY#<id>` | Subscription ID, event, status, attempt count, next attempt time, last response |
| Agreement mapping | `PK=AGREEMENT#<id>` | Opportunity ID |
| Processed event | `PK=EVENT#<eventId>` | Expires by TTL |

A global secondary index on delivery status and next attempt time serves the scheduler's query.

## External calls

| System | Call | Authentication |
|---|---|---|
| Salesforce REST API | Update Opportunity | OAuth JWT bearer flow through a connected app, with the private key held as an SST secret |
| Subscriber URLs | `POST` event | HMAC signature |

Salesforce setup needed: the two custom fields on Opportunity, a connected app with a certificate, and an integration user pre-authorised for that app.

## Done when

- Each stage of an agreement updates `Agreement_Status__c` on the Opportunity within a few seconds.
- A registered subscriber receives each event with a signature that verifies against its secret.
- A subscriber that returns `500` is retried on the backoff schedule, and the delivery is marked `FAILED` after the last attempt.
- Restarting the task mid-retry loses no delivery.
- Replaying a failed delivery after the subscriber recovers succeeds.
- Stopping and restarting the task resumes from committed offsets with no skipped events and no duplicate Salesforce updates.
