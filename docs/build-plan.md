# Build plan

Nine steps. Each ends with something you can demonstrate, and each introduces one or two technologies. The pipeline works end to end, without Salesforce, by the end of step 5.

| Step | Builds | Introduces |
|---|---|---|
| 1 | Foundation | SST, Kafka, schema registry, Avro, OpenTelemetry, GitHub Actions |
| 2 | Agreement API and create form | NestJS on Lambda, API Gateway, DynamoDB, the outbox, React |
| 3 | Document Generator | Lambda Kafka consumers, idempotency, dead-lettering, S3 |
| 4 | Signing Service | DocuSign, inbound webhooks |
| 5 | Vault Service | S3 Object Lock, presigned URLs |
| 6 | Read model and timeline | Event projections |
| 7 | Notifier | NestJS Kafka microservice, Fargate, outbound webhooks |
| 8 | Salesforce | Flows, connected apps, JWT bearer flow |
| 9 | Dashboards and alerts | Grafana dashboards and alerting as code |

## Step 1: Foundation

Set up everything the apps share, and prove the pipeline with an almost empty stack.

- Create the `signing-platform-apps` monorepo with pnpm workspaces, TypeScript, and Vitest.
- Create accounts: Confluent Cloud (cluster and schema registry), Grafana Cloud, DocuSign developer, Salesforce Developer Edition.
- Write the SST config with the shared API Gateway and the four Kafka topics.
- Write the shared package: the `.avsc` files, type generation, Avro encode and decode with the registry client, the outbox writer and relay, the idempotent consumer wrapper, the logger, and OpenTelemetry setup including trace context in the outbox and Kafka headers.
- Write the Docker Compose file for local Kafka, schema registry, DynamoDB Local, and Grafana.
- Set up the AWS OIDC role and both GitHub Actions workflows. See [CI/CD](ci-cd.md).

**Done when:** a merge to `main` deploys the `demo` stage; the topics and schema subjects exist; a throwaway Lambda publishes one Avro event through the outbox and its trace appears in Grafana, crossing the outbox hop unbroken.

## Step 2: Agreement API and create form

Spec: [Agreement API](apps/agreement-api.md), [Dashboard](apps/dashboard.md).

Build the write side only: `POST /agreements`, the agreements table, and the outbox relay. Build the dashboard's create form. Leave the read model for step 6; `GET` routes can return data from the agreements table for now.

**Done when:** submitting the form puts an `AgreementRequested` event on the topic, and a repeated idempotency key produces no second event.

## Step 3: Document Generator

Spec: [Document Generator](apps/document-generator.md).

The first Lambda consumer. Expect to spend time on the event source mapping, SASL credentials, and Avro decoding; that work is reused by every later service.

**Done when:** the criteria in the spec pass, including the replay and dead-letter cases.

## Step 4: Signing Service

Spec: [Signing Service](apps/signing-service.md).

Set up the DocuSign integration key, JWT consent, and Connect webhook first, and test them with a script before writing the service.

**Done when:** you receive a signing email for an agreement created in the dashboard, and signing it produces `EnvelopeCompleted`.

## Step 5: Vault Service

Spec: [Vault Service](apps/vault-service.md).

Create the vault bucket with Object Lock from the start; it cannot be enabled later.

**Done when:** a signed agreement ends up in the vault with a retention date, and the download route returns it. Add the end-to-end smoke script to the deploy workflow at this point.

## Step 6: Read model and timeline

Spec: [Agreement API](apps/agreement-api.md), [Dashboard](apps/dashboard.md).

Add the consumer on all four topics and the timeline table. Switch the `GET` routes to it. Build the list and detail screens.

**Done when:** the detail screen shows an agreement's timeline advancing through every stage, and the download buttons work.

## Step 7: Notifier

Spec: [Notifier](apps/notifier.md).

Build outbound webhooks first and leave Salesforce write-back for step 8. Use a free request-inspection service or a small local server as the test subscriber.

**Done when:** the webhook criteria in the spec pass, including retry, failure, replay, and restart.

## Step 8: Salesforce

Spec: [Agreement API](apps/agreement-api.md) (inbound), [Notifier](apps/notifier.md) (outbound).

- Inbound: the Named Credential and the record-triggered Flow on Opportunity.
- Outbound: the custom fields, the connected app, and the write-back in the Notifier.

**Done when:** moving an Opportunity to Closed Won in Salesforce leads, with no other action except signing, to `Agreement_Status__c` reading `VAULTED`.

## Step 9: Dashboards and alerts

Spec: [Observability](observability.md).

All services now emit data. Build the two dashboards and the alert rules as JSON, and add provisioning to the deploy workflow.

**Done when:** both dashboards show live data on `demo`, a deliberately poisoned message fires the dead-letter alert, and one agreement can be followed from request to vault in Grafana using only its correlation ID.

## Optional extensions

- Port Kafka from Confluent Cloud to AWS MSK, to learn VPC networking and IAM authentication for Kafka.
- Embedded signing inside the dashboard.
- Real authentication with Cognito and roles.
- A second contract template with a schema change, to exercise `BACKWARD` compatibility for real.
