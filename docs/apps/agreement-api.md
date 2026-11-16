# Agreement API

## Purpose

The front door. It accepts agreement requests from Salesforce and the dashboard, and it serves the agreement timeline that the dashboard displays.

## Runs as

- A NestJS HTTP application on Lambda behind API Gateway.
- An outbox relay Lambda on the agreements table's stream.
- A Lambda consumer (NestJS standalone context) on all four topics, which maintains the read model.

## HTTP routes

All routes require a Cognito access token in the `Authorization: Bearer` header. The gateway rejects a missing or invalid token with `401`; the shared permission guard rejects a token without the route's permission with `403`. See [authentication and authorisation](../architecture.md#authentication-and-authorisation).

| Route | Permission | Behaviour |
|---|---|---|
| `POST /agreements` | `agreements.write` | Validates the request, creates the agreement, returns `202` with the agreement ID |
| `GET /agreements` | `agreements.read` | Lists agreements, newest first, with current status |
| `GET /agreements/{id}` | `agreements.read` | Returns one agreement with its full event timeline |

`POST /agreements` requires an `Idempotency-Key` header. Salesforce sends the Opportunity ID; the dashboard sends a generated UUID. A repeated key returns the original agreement and publishes nothing.

## Events

| Direction | Topic | Event |
|---|---|---|
| Produces | `agreements` | `AgreementRequested` |
| Consumes | `agreements`, `documents`, `envelopes`, `vault` | all, for the read model |

## Data

Two tables. The API never reads the timeline table to make decisions; it is a projection only.

**Agreements table** (write side)

| Item | Key | Contents |
|---|---|---|
| Agreement | `PK=AGREEMENT#<id>` | Request fields, created time |
| Idempotency record | `PK=IDEMPOTENCY#<key>` | Agreement ID |
| Outbox item | `PK=OUTBOX#<eventId>` | Serialised event, topic, key, trace context |

**Timeline table** (read side)

| Item | Key | Contents |
|---|---|---|
| Agreement summary | `PK=AGREEMENT#<id>`, `SK=SUMMARY` | Status, customer, amount, last updated |
| Timeline entry | `PK=AGREEMENT#<id>`, `SK=EVENT#<occurredAt>#<eventId>` | Event type and display fields |

A global secondary index on the summary items, sorted by creation time, serves `GET /agreements`.

## External calls

None. Salesforce calls this service; this service does not call Salesforce.

## Salesforce side

A record-triggered Flow on Opportunity fires when the stage changes to Closed Won. It makes an HTTP callout to `POST /agreements` through a Named Credential, with the Opportunity ID as the idempotency key.

The Named Credential uses an External Credential set up for the OAuth client credentials flow. It holds the client ID and secret of the `salesforce` app client, fetches an access token from the Cognito token endpoint, and renews it when it expires. That client is granted only `agreements.write`, so Salesforce can create agreements and nothing else.

## Done when

- A `POST /agreements` call returns `202` and an `AgreementRequested` event appears on the `agreements` topic, Avro-encoded with a registered schema.
- Sending the same idempotency key twice produces one agreement and one event.
- Killing the handler between the DynamoDB write and the Kafka publish loses no event (the relay publishes it).
- `GET /agreements/{id}` shows a timeline entry for every event the pipeline has produced for that agreement.
- Closing an Opportunity in Salesforce creates an agreement with no manual step.
- A request with no token, an expired token, or a token from another user pool returns `401`.
- A `GET /agreements` call with the `salesforce` client's token returns `403`.
