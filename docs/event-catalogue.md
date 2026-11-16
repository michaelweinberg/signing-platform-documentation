# Event catalogue

## Topics

One topic per producing service. Only the owner writes to a topic. Every message is keyed by agreement ID, so all events for one agreement stay in order within a topic.

| Topic | Owner | Event types | Partitions |
|---|---|---|---|
| `agreements` | Agreement API | `AgreementRequested` | 3 |
| `documents` | Document Generator | `DocumentGenerated`, `DocumentGenerationFailed` | 3 |
| `envelopes` | Signing Service | `EnvelopeSent`, `EnvelopeCompleted`, `EnvelopeDeclined` | 3 |
| `vault` | Vault Service | `DocumentVaulted` | 3 |

Topics are declared in the SST config and created by the deploy pipeline.

## Schema rules

| Rule | Value |
|---|---|
| Format | Avro, Confluent wire format (magic byte plus schema ID) |
| Registry | Confluent Cloud Schema Registry |
| Subject naming | `TopicRecordNameStrategy`: `<topic>-<namespace>.<RecordName>` |
| Compatibility | `BACKWARD` |
| Source of truth | `.avsc` files in the monorepo's shared package |
| TypeScript types | Generated from the `.avsc` files; never written by hand |
| Registration | By the deploy pipeline, before the apps deploy. Producers do not auto-register |

`TopicRecordNameStrategy` is required because a topic carries more than one event type. Example subject: `envelopes-signingplatform.envelopes.EnvelopeCompleted`.

`BACKWARD` compatibility means a new schema can read data written with the previous one. In practice: new fields need a default, and fields cannot be removed unless they had a default. Deploy consumers before producers when a schema changes.

## Common fields

Every event record starts with the same fields.

| Field | Avro type | Meaning |
|---|---|---|
| `eventId` | `string` (UUID) | Unique per event. Consumers use it for idempotency |
| `eventType` | `string` | For example `envelope.completed` |
| `eventVersion` | `int` | Starts at 1; raised on any schema change |
| `occurredAt` | `long` (`timestamp-millis`) | When the producing service recorded the change |
| `agreementId` | `string` (UUID) | Also the Kafka message key |
| `correlationId` | `string` | Business identifier carried through the whole flow and into logs and spans |

Trace context is not in the event body. It travels in Kafka message headers (`traceparent`, `tracestate`). See [observability](observability.md).

## Events

### `AgreementRequested` (topic `agreements`)

Published when the Agreement API accepts a new agreement.

| Field | Type | Notes |
|---|---|---|
| `source` | enum `SALESFORCE`, `DASHBOARD` | |
| `opportunityId` | `string`, nullable | Salesforce Opportunity ID; null for dashboard requests |
| `templateId` | `string` | Which contract template to render |
| `customerName` | `string` | |
| `amount` | `bytes` (`decimal`, precision 12, scale 2) | |
| `currency` | `string` | ISO 4217 |
| `signerName` | `string` | |
| `signerEmail` | `string` | |

### `DocumentGenerated` (topic `documents`)

| Field | Type | Notes |
|---|---|---|
| `documentId` | `string` | |
| `bucket` | `string` | Drafts bucket |
| `key` | `string` | Object key of the draft PDF |
| `sha256` | `string` | Hash of the draft |
| `signerName` | `string` | Copied from the request so the Signing Service needs no lookup |
| `signerEmail` | `string` | |

### `DocumentGenerationFailed` (topic `documents`)

| Field | Type | Notes |
|---|---|---|
| `reason` | `string` | Human-readable cause, for the timeline |

### `EnvelopeSent` (topic `envelopes`)

| Field | Type | Notes |
|---|---|---|
| `envelopeId` | `string` | DocuSign envelope ID |
| `sentAt` | `long` (`timestamp-millis`) | |

### `EnvelopeCompleted` (topic `envelopes`)

| Field | Type | Notes |
|---|---|---|
| `envelopeId` | `string` | |
| `completedAt` | `long` (`timestamp-millis`) | |
| `bucket` | `string` | Signed-staging bucket |
| `signedDocumentKey` | `string` | The signed PDF |
| `certificateKey` | `string` | DocuSign's certificate of completion |

### `EnvelopeDeclined` (topic `envelopes`)

| Field | Type | Notes |
|---|---|---|
| `envelopeId` | `string` | |
| `declinedAt` | `long` (`timestamp-millis`) | |
| `reason` | `string`, nullable | The signer's stated reason, if given |

### `DocumentVaulted` (topic `vault`)

| Field | Type | Notes |
|---|---|---|
| `vaultRecordId` | `string` | |
| `signedDocumentSha256` | `string` | |
| `certificateSha256` | `string` | |
| `retainUntil` | `long` (`timestamp-millis`) | Object Lock retention date |
| `vaultedAt` | `long` (`timestamp-millis`) | |

## Agreement status

The read model and the Notifier derive one status from the events.

| Status | Set by |
|---|---|
| `REQUESTED` | `AgreementRequested` |
| `GENERATED` | `DocumentGenerated` |
| `GENERATION_FAILED` | `DocumentGenerationFailed` |
| `SENT` | `EnvelopeSent` |
| `SIGNED` | `EnvelopeCompleted` |
| `DECLINED` | `EnvelopeDeclined` |
| `VAULTED` | `DocumentVaulted` |

Because events arrive from four topics, the read model can receive them out of order across topics. It stores every event in the timeline and sets the status to the furthest stage seen, never moving backwards.
