# Idempotency and Upsert Strategy

This document defines how the synchronization workflow prevents duplicate records and safely reprocesses previously seen data.

---

## Objectives

The synchronization strategy must:

- avoid creating duplicate CRM records
- support safe workflow retries
- allow the same source record to be processed more than once without side effects
- update existing CRM records when source data changes
- preserve the newest valid version of each record

---

## Idempotency Key

Each source record must contain a stable external identifier.

Source field:

`customerId`

Normalized field:

`externalId`

This field is used as the primary idempotency key.

Example:

    customerId: CUS-1001

becomes:

    externalId: CUS-1001

The value must remain stable across synchronization runs.

---

## Upsert Strategy

The CRM operation should follow this logic:

    Receive normalized record
            ↓
    Search by externalId
            ↓
    ┌─────────────────────┐
    │ Record exists?      │
    └─────────────────────┘
        ↓ Yes       ↓ No
      Update       Create
        ↓             ↓
        └──────┬──────┘
               ↓
          Mark success

This approach allows the workflow to safely process the same source record multiple times.

---

## Duplicate Prevention

The workflow must not use name, email or phone as the primary deduplication key.

These fields can change or may not be unique.

Examples of unsafe identifiers:

- full name
- phone number
- email address

Preferred identifier:

- stable external system ID

---

## Update Decision

When a CRM record already exists, the workflow should compare source timestamps before updating.

Source field:

`sourceUpdatedAt`

Recommended logic:

    incoming sourceUpdatedAt
            ↓
    compare with stored timestamp
            ↓
    newer?
      ↓ Yes      ↓ No
    update      ignore

This prevents older source data from overwriting newer CRM information.

---

## Safe Reprocessing

A previously processed record may be received again because of:

- workflow retry
- API pagination restart
- scheduled synchronization
- manual reprocessing
- source system replay

The workflow should safely process the record again without creating duplicates.

Example:

    First run:
    CUS-1001 → Create CRM record

    Second run:
    CUS-1001 → Update existing record

    Third run with unchanged data:
    CUS-1001 → No duplicate created

---

## Failure Recovery

If the CRM operation fails after the source record has already been validated, the same record should be safe to retry.

Example:

    Validate
       ↓
    Normalize
       ↓
    CRM upsert
       ↓
    HTTP 503
       ↓
    Retry
       ↓
    CRM upsert

Because the operation is based on `externalId`, retries should not create duplicate records.

---

## Idempotency Metadata

Recommended processing metadata:

    {
      "externalId": "CUS-1001",
      "sourceUpdatedAt": "2026-08-14T10:15:00-03:00",
      "processedAt": "2026-08-14T14:30:00-03:00",
      "operation": "update",
      "status": "success"
    }

This information can support troubleshooting and observability.

---

## Edge Cases

### Missing External ID

Records without a stable external identifier must not be sent to the CRM.

Expected behavior:

- mark as `missing_external_id`
- send to error handling
- do not attempt CRM creation

---

### Duplicate External IDs

If multiple source records contain the same external ID within the same batch, the workflow should detect the condition before CRM processing.

Recommended behavior:

- keep the newest record
- flag the duplicate condition
- avoid multiple conflicting updates

---

### Older Source Record

If the incoming record is older than the CRM version, the workflow should not overwrite newer information.

Expected behavior:

- skip the update
- record the reason
- mark the operation as ignored or stale

---

## Design Principle

Idempotency should be treated as a core integration requirement, not as an optional optimization.

A synchronization workflow must be safe to retry, restart and reprocess without producing duplicate records or inconsistent state.
