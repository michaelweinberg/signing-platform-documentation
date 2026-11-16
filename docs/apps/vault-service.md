# Vault Service

## Purpose

Stores signed documents so they cannot be altered or deleted during their retention period, and provides the only way to download them.

## Runs as

- A Lambda consumer (NestJS standalone context) on the `envelopes` topic.
- A Lambda HTTP handler for the download route.
- An outbox relay Lambda on the vault table's stream.

## Events

| Direction | Topic | Event |
|---|---|---|
| Consumes | `envelopes` | `EnvelopeCompleted` (ignores the others) |
| Produces | `vault` | `DocumentVaulted` |

## Behaviour

### Vaulting

1. Decode the event and check the event ID has not been processed.
2. Copy the signed PDF and the certificate from the signed-staging bucket into the vault bucket, with an Object Lock retain-until date.
3. Compute the SHA-256 hash of each file.
4. In one transaction, write the vault record, the processed event ID, and the `DocumentVaulted` outbox item.

Object keys are derived from the agreement ID (`vault/<agreementId>/signed.pdf`, `vault/<agreementId>/certificate.pdf`). A retry writes a new version of the same key with the same content; the vault record points at the first version ID.

### Downloading

`GET /agreements/{id}/documents/{kind}/download`, where `kind` is `signed` or `certificate`. Requires a Cognito access token with the `documents.download` permission, which both user groups have and no machine client has. See [authentication and authorisation](../architecture.md#authentication-and-authorisation).

Returns a presigned S3 URL that expires after 5 minutes. The response also includes the recorded SHA-256 hash so the caller can verify the file.

## The vault bucket

| Setting | Value | Why |
|---|---|---|
| Versioning | On | Required by Object Lock |
| Object Lock | Governance mode | Blocks deletion and overwrite for normal roles, but a privileged role can still remove test data. Compliance mode cannot be undone by anyone, including the account root, which would block teardown |
| Default retention | 1 year | Short enough for a learning build |
| Public access | Blocked | |
| Encryption | S3-managed keys | |

Object Lock must be enabled when the bucket is created; it cannot be added later.

## Data

**Vault table**

| Item | Key | Contents |
|---|---|---|
| Vault record | `PK=AGREEMENT#<id>`, `SK=VAULT` | Keys, version IDs, hashes, retain-until date, vaulted time |
| Processed event | `PK=EVENT#<eventId>` | Expires by TTL |
| Outbox item | `PK=OUTBOX#<eventId>` | Serialised event, topic, key, trace context |

## External calls

None. The Signing Service has already fetched the files from DocuSign.

## Done when

- An `EnvelopeCompleted` event results in both files in the vault bucket with a retention date, and a `DocumentVaulted` event whose hashes match the files.
- An attempt to delete a vaulted object version with the service's own role is denied.
- The download route returns a working URL that stops working after expiry.
- A download request without a token returns `401`, and one with the `salesforce` client's token returns `403`.
- Replaying `EnvelopeCompleted` produces no second vault record and no second event.
