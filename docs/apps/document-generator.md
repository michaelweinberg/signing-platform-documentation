# Document Generator

## Purpose

Turns an agreement request into a contract PDF and stores the draft.

This is the first Lambda consumer in the build order, so the shared consumer code (Avro decoding, idempotency, dead-lettering) is written here and reused by every later service.

## Runs as

- A Lambda consumer (NestJS standalone context) on the `agreements` topic.
- An outbox relay Lambda on the documents table's stream.

## Events

| Direction | Topic | Event |
|---|---|---|
| Consumes | `agreements` | `AgreementRequested` |
| Produces | `documents` | `DocumentGenerated`, `DocumentGenerationFailed` |

## Behaviour

1. Decode the event and check the event ID has not been processed.
2. Select the template named by `templateId`.
3. Render the PDF with `@react-pdf/renderer`. Templates are React components in this app's package.
4. Compute the SHA-256 hash and write the PDF to the drafts bucket at `drafts/<agreementId>/<documentId>.pdf`.
5. In one transaction, write the document record, the processed event ID, and the `DocumentGenerated` outbox item.

An unknown template or invalid data is a permanent failure: the service publishes `DocumentGenerationFailed` and does not retry. Any other error is retried, then dead-lettered.

The S3 write happens before the transaction. If the transaction fails, the retry overwrites the same key, so no orphaned state results.

## Data

**Documents table**

| Item | Key | Contents |
|---|---|---|
| Document | `PK=AGREEMENT#<id>`, `SK=DOCUMENT#<documentId>` | Template, bucket, key, hash, created time |
| Processed event | `PK=EVENT#<eventId>` | Expires by TTL after 30 days |
| Outbox item | `PK=OUTBOX#<eventId>` | Serialised event, topic, key, trace context |

**Drafts bucket**: private, with a lifecycle rule that deletes drafts after 90 days. The signed copy in the vault is the record; drafts are working files.

## External calls

None.

## Done when

- An `AgreementRequested` event results in a readable PDF in the drafts bucket and a `DocumentGenerated` event whose hash matches the file.
- Replaying the same event produces no second document and no second event.
- An event with an unknown template produces `DocumentGenerationFailed` and nothing in the dead-letter queue.
- A forced runtime error lands the message in the dead-letter queue after retries.
