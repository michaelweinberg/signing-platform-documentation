# signing-platform-documentation

Design documentation for the signing platform: a learning build that mirrors a document generation, signing, and vaulting system on React, NestJS, SST, AWS Lambda, API Gateway, DynamoDB, Kafka, DocuSign, and Salesforce.

This repo holds documentation only. The code lives in a sibling SST monorepo, `signing-platform-apps`.

## The scenario

A Salesforce Opportunity reaches Closed Won. A contract PDF is generated from the Opportunity data, sent for signature through DocuSign, and the signed copy is stored in an immutable vault. The status is written back to Salesforce at each stage.

## Where to start

| Document | What it covers |
|---|---|
| [Architecture](docs/architecture.md) | The six apps, how they connect, and the shared rules every service follows |
| [Event catalogue](docs/event-catalogue.md) | Topics, event types, Avro schemas, and schema rules |
| [Build plan](docs/build-plan.md) | The order to build in, with "done when" criteria per step |
| [Observability](docs/observability.md) | OpenTelemetry, Grafana, trace propagation, dashboards, alerts |
| [CI/CD](docs/ci-cd.md) | GitHub Actions workflows, credentials, and deploy gates |
| [Decisions](docs/decisions.md) | Each design decision, the alternative rejected, and why |

App specs:

| App | Spec |
|---|---|
| Agreement API | [docs/apps/agreement-api.md](docs/apps/agreement-api.md) |
| Document Generator | [docs/apps/document-generator.md](docs/apps/document-generator.md) |
| Signing Service | [docs/apps/signing-service.md](docs/apps/signing-service.md) |
| Vault Service | [docs/apps/vault-service.md](docs/apps/vault-service.md) |
| Notifier | [docs/apps/notifier.md](docs/apps/notifier.md) |
| Dashboard | [docs/apps/dashboard.md](docs/apps/dashboard.md) |

The original request is in [client-brief.md](client-brief.md).
