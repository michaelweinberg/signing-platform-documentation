# Architecture

## Purpose and limits

This is a learning build. It is designed to exercise every technology in the brief in a realistic way, and to be finishable by one person working part time over a few weeks. It is not production hardened: there is no multi-region setup, no compliance work, and no user authentication beyond a shared API key.

It uses real external systems: a DocuSign developer sandbox, a Salesforce Developer Edition org, and a real AWS account.

## System overview

```mermaid
flowchart LR
    SF[Salesforce]
    DS[DocuSign]
    SUB[Webhook subscribers]
    UI[Dashboard<br/>React]

    subgraph AWS
        API[Agreement API<br/>Nest HTTP on Lambda]
        GEN[Document Generator<br/>Nest Lambda consumer]
        SIGN[Signing Service<br/>Nest Lambda consumer + webhook]
        VAULT[Vault Service<br/>Nest Lambda consumer]
        NOTE[Notifier<br/>Nest microservice on Fargate]
    end

    subgraph Confluent Cloud
        T1([agreements])
        T2([documents])
        T3([envelopes])
        T4([vault])
    end

    SF -- Flow HTTP callout --> API
    UI -- create / read --> API
    API --> T1
    T1 --> GEN --> T2
    T2 --> SIGN --> T3
    SIGN -- create envelope --> DS
    DS -- Connect webhook --> SIGN
    T3 --> VAULT --> T4
    T1 & T2 & T3 & T4 --> NOTE
    T1 & T2 & T3 & T4 --> API
    NOTE -- status update --> SF
    NOTE -- signed webhooks --> SUB
    UI -- download --> VAULT
```

## The apps

| App | Runs as | Consumes | Produces | Owns |
|---|---|---|---|---|
| [Agreement API](apps/agreement-api.md) | NestJS HTTP on Lambda behind API Gateway, plus a Lambda consumer for the read model | all four topics (read model) | `agreements` | agreements table, timeline table |
| [Document Generator](apps/document-generator.md) | NestJS Lambda consumer | `agreements` | `documents` | documents table, drafts bucket |
| [Signing Service](apps/signing-service.md) | NestJS Lambda consumer, plus a webhook route | `documents` | `envelopes` | envelopes table, signed-staging bucket |
| [Vault Service](apps/vault-service.md) | NestJS Lambda consumer, plus a download route | `envelopes` | `vault` | vault table, vault bucket |
| [Notifier](apps/notifier.md) | Long-running NestJS Kafka microservice on Fargate, plus Lambda routes for subscriptions | all four topics | nothing | notifier table |
| [Dashboard](apps/dashboard.md) | React single-page app | nothing | nothing | nothing |

## Happy path

```mermaid
sequenceDiagram
    participant SF as Salesforce
    participant API as Agreement API
    participant GEN as Document Generator
    participant SIGN as Signing Service
    participant DS as DocuSign
    participant VAULT as Vault Service
    participant NOTE as Notifier

    SF->>API: POST /agreements (Closed Won)
    API-->>GEN: AgreementRequested
    GEN-->>SIGN: DocumentGenerated
    SIGN->>DS: create envelope
    SIGN-->>NOTE: EnvelopeSent
    Note over DS: signer signs (minutes to days)
    DS->>SIGN: Connect webhook (completed)
    SIGN->>DS: fetch signed PDF + certificate
    SIGN-->>VAULT: EnvelopeCompleted
    VAULT-->>NOTE: DocumentVaulted
    NOTE->>SF: update Opportunity status
```

Dashed arrows are Kafka events. The Notifier and the Agreement API's read model consume every event; only the first relevant one is drawn.

## Rules every service follows

These apply to all five backend apps. The shared package in the monorepo provides the code for each.

### One table per service

Each service has its own DynamoDB table. No service reads another service's table. Data crosses service boundaries only through events, with two stated exceptions for files:

- The Signing Service has read access to the Document Generator's drafts bucket, at the key given in `DocumentGenerated`.
- The Vault Service has read access to the Signing Service's signed-staging bucket, at the keys given in `EnvelopeCompleted`.

### Publishing: transactional outbox

A service never writes to DynamoDB and Kafka in the same handler. It writes its state change and an outbox item in one DynamoDB transaction. A DynamoDB Streams Lambda (the outbox relay) reads new outbox items and publishes them to Kafka.

```mermaid
flowchart LR
    H[Handler] -- one transaction --> DB[(Service table<br/>state + outbox item)]
    DB -- DynamoDB Stream --> R[Outbox relay Lambda]
    R -- Avro --> K([Kafka topic])
```

The outbox item stores the serialised event, the target topic, the message key, and the trace context of the handler that wrote it.

### Consuming: idempotent, with a dead-letter queue

Kafka delivers at least once, and DocuSign resends webhooks. Every consumer:

1. Decodes the Avro payload using the schema registry client.
2. Attempts a conditional write of the event ID to its table. If the ID already exists, it stops and reports success.
3. Does its work and writes state plus any outbox item in one transaction with the event ID record.

A message that still fails after retries goes to an SQS dead-letter queue, one per consumer. Grafana alerts on any message in a dead-letter queue.

### Two consumer models

| Model | Used by | Who manages offsets |
|---|---|---|
| Lambda event source mapping | Agreement API read model, Document Generator, Signing Service, Vault Service | AWS polls the topic and invokes the function with batches |
| NestJS Kafka microservice | Notifier | The process holds its own connection and consumer group |

Lambda consumers boot NestJS as a standalone application context (modules and dependency injection, no HTTP server).

### One shared API Gateway

All HTTP routes sit on one API Gateway. Each service owns its routes:

| Route | Owner |
|---|---|
| `POST /agreements`, `GET /agreements`, `GET /agreements/{id}` | Agreement API |
| `POST /webhooks/docusign` | Signing Service |
| `GET /agreements/{id}/documents/{kind}/download` | Vault Service |
| `POST /webhook-subscriptions`, `GET /webhook-subscriptions`, `POST /webhook-deliveries/{id}/replay` | Notifier |

All routes except the DocuSign webhook require the shared API key. The DocuSign webhook is authenticated by HMAC signature.

### External systems sit behind adapters

DocuSign and Salesforce are each called through one adapter interface. Unit tests use a fake implementation. Only the Signing Service calls DocuSign. Only the Notifier calls Salesforce.

## Infrastructure

| Piece | Choice | Notes |
|---|---|---|
| Infrastructure as code | SST | One config for all apps; Kafka topics declared through the Confluent provider |
| Kafka | Confluent Cloud, entry-tier cluster | Public endpoint with SASL; the cluster is created by hand once |
| Schema registry | Confluent Cloud Schema Registry | See the [event catalogue](event-catalogue.md) |
| Kafka client | `kafkajs` | NestJS's Kafka transport is built on it; pure JavaScript, so Lambda packaging stays simple |
| Stages | personal dev, `demo` | Dev is deployed from a laptop in SST live mode; `demo` is deployed only by the pipeline |
| Secrets | SST secrets per stage | DocuSign, Salesforce, and Kafka credentials; never in the repo |
| Monorepo tooling | pnpm workspaces, Vitest | |

### Cost notes

Checked November 2026. The Confluent and Grafana figures come partly from third-party summaries and they disagree in places, so confirm on each vendor's pricing page before relying on them.

- **Confluent Cloud**: the entry cluster tier has no base charge and bills for data in, data out, and storage, all negligible at this volume. New accounts get trial credit. Schema Registry's Essentials package has no base charge and includes 100 free schemas according to Confluent's documentation. Sources disagree on how many partitions the entry tier includes, so topics use 3 partitions each.
- **Grafana Cloud**: the free plan is reported as about 10,000 active metric series, 50 GB of logs and 50 GB of traces per month, 14-day retention, and 3 users.
- **AWS**: Lambda, API Gateway, DynamoDB, and S3 are free or close to it at this scale. The Notifier's Fargate task is the one always-on cost; stop it when not in use.

## Testing

| Layer | Runs against | Purpose |
|---|---|---|
| Unit | Fakes behind the DocuSign and Salesforce adapters | Business rules in each service |
| Integration | Kafka, schema registry, and DynamoDB Local in Docker | Outbox, consumers, idempotency, Avro round trips |
| End-to-end smoke | The deployed `demo` stage | One agreement from request to vaulted; run by the pipeline after each deploy |

## To verify during the build

These were assumed in the design and not confirmed against current documentation:

- That a Lambda event source mapping for self-managed Kafka supports an SQS on-failure destination in the chosen region.
- How SST passes Confluent SASL credentials to the event source mapping (AWS expects them in Secrets Manager).
- That the Confluent provider can be used from SST for topic creation with the entry-tier cluster's API keys.
- Whether the smoke test can complete an envelope in the DocuSign sandbox without a person signing. If it cannot, the automated smoke test stops at `SENT` and the signing and vaulting stages are checked by hand.
- Whether DocuSign's HMAC key can be set per envelope or only on an account-level Connect configuration.
