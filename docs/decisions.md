# Decisions

Each entry records what was chosen, what was rejected, and why. All were settled on 2026-11-16.

## Scope

| # | Decision | Rejected | Why |
|---|---|---|---|
| 1 | This is a learning build that mirrors the client's stack | Building the production system | The goal is fluency in the technologies; production hardening is out of scope |
| 2 | This repo holds docs only; code goes in a sibling monorepo | Docs and code together | Keeps the design readable on its own |
| 3 | One scenario: Opportunity Closed Won, generate, sign, vault, write back | Several document types or flows | One story exercises every stage and both integrations in both directions |
| 4 | Real DocuSign and Salesforce sandboxes | Local mocks | The auth flows and webhooks are the hard parts, and mocks teach none of them |
| 5 | Deploy to a real AWS account | Local emulation only | Cheap at this scale, and SST live mode is part of what is being learned |
| 6 | Four to six small apps, solo, a few weeks part time | A larger system | Small enough to finish |

## System shape

| # | Decision | Rejected | Why |
|---|---|---|---|
| 7 | Six apps, one per lifecycle stage, plus Notifier and Dashboard | Fewer, larger services | Each app owns one job and its own data; the Notifier is the single outbound point |
| 8 | Kafka on Confluent Cloud | AWS MSK | MSK bills hourly when idle and needs VPC and NAT setup. Trade-off: no MSK-specific networking or IAM auth is learned; porting is an optional extension |
| 9 | Salesforce triggers by a Flow HTTP callout | Subscribing to Platform Events or Change Data Capture | Subscribing needs a long-lived gRPC connection, which Lambda cannot hold |
| 10 | Vault is S3 with versioning and Object Lock in governance mode, hashes in DynamoDB, presigned downloads | Compliance mode | Compliance mode cannot be undone, which would block deleting test data and teardown |
| 11 | PDFs rendered with `@react-pdf/renderer` | Headless Chromium on Lambda; DocuSign templates | No browser binary to package; generation stays our own service |
| 12 | The Notifier also sends outbound HMAC-signed webhooks | Inbound webhooks only | Receiving and delivering webhooks are different skills |
| 13 | One SST monorepo for all apps | One repo per app | One deploy, one home for event contracts, less overhead for one person |
| 14 | Dashboard has a create form and a shared API key | Cognito and roles | The form allows demos without Salesforce; real auth is a distraction here |

## Messaging and data

| # | Decision | Rejected | Why |
|---|---|---|---|
| 15 | Lambda consumers for four apps; the Notifier is a long-running NestJS Kafka microservice | One model for all | The two operating models differ, and the brief asks for both |
| 16 | NestJS in every backend app, including standalone context in Lambda consumers | Plain handlers | One consistent structure. Cost: slower cold starts |
| 17 | Transactional outbox through DynamoDB Streams | Writing to DynamoDB and Kafka in one handler | A dual write loses or duplicates events when one side fails |
| 18 | One topic per producing service, keyed by agreement ID | One topic per event type | Clear ownership, and ordering is kept between related events |
| 19 | Avro with Confluent Schema Registry | JSON with Zod schemas | Chosen by the project owner, to match common Kafka practice |
| 20 | `TopicRecordNameStrategy`, `BACKWARD` compatibility, schemas registered at deploy | Default subject naming; producer auto-registration | Several event types share a topic; auto-registration can push a breaking schema by accident |
| 21 | Idempotent consumers and SQS dead-letter queues | Relying on exactly-once delivery | Kafka and DocuSign both redeliver |
| 22 | One private DynamoDB table per service; the dashboard reads a projection in the Agreement API | Shared tables; the dashboard calling every service | Keeps services decoupled; one query serves the timeline |
| 23 | DocuSign remote signing with a Connect webhook | Embedded signing | The common case, and the webhook stays the only source of the outcome |

## Operations

| # | Decision | Rejected | Why |
|---|---|---|---|
| 24 | OpenTelemetry for traces, metrics, and logs, to Grafana Cloud | Structured logs and CloudWatch alarms only | Chosen by the project owner |
| 25 | Lambda telemetry through the collector layer; a sidecar for the Notifier | Exporting directly from function code | The layer flushes outside the handler, so spans are not lost on freeze |
| 26 | Trace context saved in outbox items and Kafka headers; a span link across the signing wait | One continuous trace | The trace would break at each stream hop; signing can take days |
| 27 | Alerts and dashboards in Grafana, stored as JSON in the apps repo | CloudWatch alarms; hand-built dashboards | One place for both; they survive teardown |
| 28 | OpenTelemetry and the pipeline are set up in the foundation step | Adding them late | Retrofitting context propagation is much more work |
| 29 | Two stages: personal dev and `demo`; SST secrets | More environments | Enough for one person |
| 30 | Three test layers: unit with fakes, integration in Docker, end-to-end smoke | End-to-end only | Fast local feedback plus one proof per deploy |
| 31 | Salesforce write-back with the JWT bearer flow | Username-password flow | Server to server, and mirrors the DocuSign JWT grant |
| 32 | Webhook deliveries persisted, retried with backoff, then failed with manual replay | In-memory retries | Retries survive a restart |

## CI/CD

| # | Decision | Rejected | Why |
|---|---|---|---|
| 33 | GitHub Actions: checks on pull requests, deploy to `demo` on merge to `main` | Deploying from a laptop only; per-pull-request stages | Chosen by the project owner. Per-PR stages would each need their own topics and subjects |
| 34 | AWS access by OIDC role | Stored access keys | No long-lived credentials exist |
| 35 | Integration credentials stay as SST secrets; the pipeline holds only its own | All secrets in GitHub | The pipeline never needs to see them |
| 36 | Schema compatibility checked on pull requests; schemas registered before deploy | Checking at deploy only | A breaking change is caught before merge |
| 37 | Kafka topics declared in SST; the cluster created by hand | Topics created by hand | Topics version with the code that uses them |
| 38 | Smoke test after deploy; fix forward on failure | Automatic rollback; manual approval | Rollback across Lambdas, schemas, and topics is hard to do correctly at this scale |

## Design choices made while writing the specs

These were not discussed in the design session. They follow from the decisions above, and are open to change.

| # | Choice | Why |
|---|---|---|
| A | The Signing Service downloads the signed PDF and certificate into a staging bucket; the Vault Service copies from there | Keeps DocuSign credentials and calls in one service |
| B | The Notifier's HTTP routes are Lambda handlers sharing its table, not routes on the Fargate task | Avoids the cost of a load balancer |
| C | The vault download route belongs to the Vault Service, on the shared API Gateway | The vault table is private, so no other service can issue the URL |
| D | The read model sets status to the furthest stage seen | Events from different topics can arrive out of order |
| E | Topics use 3 partitions each | Sources disagree on the entry tier's partition allowance |
