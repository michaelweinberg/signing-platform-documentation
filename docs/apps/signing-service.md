# Signing Service

## Purpose

Sends generated documents for signature through DocuSign and reports the outcome. It is the only service that calls DocuSign.

## Runs as

- A Lambda consumer (NestJS standalone context) on the `documents` topic.
- A Lambda HTTP handler for `POST /webhooks/docusign`.
- An outbox relay Lambda on the envelopes table's stream.

## Events

| Direction | Topic | Event |
|---|---|---|
| Consumes | `documents` | `DocumentGenerated` (ignores `DocumentGenerationFailed`) |
| Produces | `envelopes` | `EnvelopeSent`, `EnvelopeCompleted`, `EnvelopeDeclined` |

## Behaviour

### Sending

1. Decode the event and check the event ID has not been processed.
2. Read the draft PDF from the drafts bucket and verify its hash against the event.
3. Create a DocuSign envelope with one signer (remote signing: DocuSign emails the signer).
4. In one transaction, write the envelope record (including the current trace context), the processed event ID, and the `EnvelopeSent` outbox item.

If step 4 fails after step 3 succeeded, the retry would create a second envelope. To prevent this, the service writes a `PENDING` envelope record keyed by agreement ID before calling DocuSign, and a retry that finds a `PENDING` record looks the envelope up in DocuSign before creating another.

### Receiving the webhook

DocuSign Connect calls `POST /webhooks/docusign` when an envelope is completed or declined.

1. Verify the HMAC signature header against the raw request body. Reject with `401` on mismatch.
2. Look up the envelope record by DocuSign envelope ID. If the outcome is already recorded, return `200` and stop (DocuSign resends).
3. For a completed envelope, download the signed PDF and the certificate of completion from DocuSign into the signed-staging bucket.
4. In one transaction, update the envelope record and write the `EnvelopeCompleted` or `EnvelopeDeclined` outbox item.
5. Return `200`. Any failure returns `5xx` so DocuSign retries.

The webhook starts a new trace, linked to the trace saved on the envelope record. See [observability](../observability.md).

## Data

**Envelopes table**

| Item | Key | Contents |
|---|---|---|
| Envelope | `PK=AGREEMENT#<id>`, `SK=ENVELOPE` | DocuSign envelope ID, status, timestamps, saved trace context |
| Processed event | `PK=EVENT#<eventId>` | Expires by TTL |
| Outbox item | `PK=OUTBOX#<eventId>` | Serialised event, topic, key, trace context |

A global secondary index on DocuSign envelope ID serves the webhook lookup.

**Signed-staging bucket**: private, objects deleted after 30 days. The Vault Service copies from here.

## External calls

| System | Call | Authentication |
|---|---|---|
| DocuSign eSignature API | Create envelope, get envelope, download documents | OAuth JWT grant, with the private key held as an SST secret |

DocuSign setup needed in the developer sandbox: an integration key with an RSA key pair, one-time user consent for the JWT grant, and a Connect configuration (or per-envelope notification settings) pointing at the webhook URL with an HMAC key.

## Done when

- A `DocumentGenerated` event results in a signing email arriving and an `EnvelopeSent` event.
- Signing in the DocuSign sandbox results in the signed PDF and certificate in the staging bucket and an `EnvelopeCompleted` event.
- Declining results in `EnvelopeDeclined`.
- A webhook with a bad signature is rejected with `401` and changes nothing.
- The same webhook delivered twice produces one event.
- Replaying `DocumentGenerated` creates no second envelope.
