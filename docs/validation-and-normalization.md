# Validation and Normalization Rules

This document describes the data validation and normalization rules applied before records are sent to the CRM.

---

## Objectives

The validation layer must:

- prevent malformed data from breaking the synchronization workflow
- standardize fields before CRM ingestion
- preserve valid records even when optional fields are invalid
- create clear validation states for observability and troubleshooting

---

## Field Rules

### External ID

Source field:

`customerId`

Target field:

`externalId`

Rules:

- required
- must be unique
- used as the primary reference for idempotent upserts
- records without an external ID must not be processed

---

### Name

Source field:

`name`

Target field:

`fullName`

Rules:

- trim leading and trailing spaces
- preserve the original spelling
- reject only when the field is completely empty

---

### Email

Source field:

`email`

Rules:

- convert empty strings to `null`
- trim whitespace
- convert to lowercase
- validate basic email structure
- invalid emails must not stop the entire workflow
- when invalid, store `null`
- flag the record with `validationStatus = invalid_email`

---

### Phone

Source field:

`phone`

Rules:

- remove spaces
- remove punctuation
- remove the leading `+`
- preserve country code
- store only numeric characters

Example:

    +55 (31) 99999-1001

becomes:

    5531999991001

---

### Status

Source values:

- `active`
- `inactive`

Target values:

- `ACTIVE`
- `INACTIVE`

Rules:

- normalize values to uppercase
- unknown values must be flagged for review

---

### Updated At

Source field:

`updatedAt`

Target field:

`sourceUpdatedAt`

Rules:

- preserve timezone information
- store in ISO 8601 format
- use the timestamp to determine which version of a record is newer

---

## Validation Status

Each record must receive a validation status.

Possible values:

- `valid`
- `invalid_email`
- `invalid_phone`
- `missing_external_id`
- `invalid_status`
- `rejected`

---

## Failure Strategy

Validation failures must be handled at record level whenever possible.

A malformed optional field should not interrupt the entire synchronization batch.

Example:

    Batch with 100 records
            ↓
    98 valid
    2 invalid email
            ↓
    98 processed normally
    2 processed without email and flagged

Critical validation failures, such as a missing external identifier, should route the record to the error handling workflow.

---

## Record-Level Processing

The validation layer should isolate errors per record rather than treating the entire batch as invalid.

Example:

    Input batch
        ↓
    Record validation
        ↓
    ┌────────────────────┐
    │ Valid record       │ → Continue processing
    │ Invalid optional   │ → Normalize + flag
    │ Critical failure   │ → Error handling
    └────────────────────┘

This approach improves resilience and prevents one malformed record from blocking unrelated valid records.

---

## Normalization Flow

The normalization stage should transform source fields into the target CRM format before any upsert operation.

Example:

    External API record
            ↓
    Trim values
            ↓
    Normalize phone
            ↓
    Normalize email
            ↓
    Normalize status
            ↓
    Assign validation status
            ↓
    CRM-ready payload

---

## Data Quality Principles

The synchronization process should follow these principles:

- never assume source data is perfectly formatted
- validate required identifiers before processing
- normalize fields consistently
- preserve useful data whenever possible
- avoid discarding an entire record because of a non-critical invalid field
- isolate critical failures for investigation
- never expose sensitive information in logs

---

## Design Principle

The synchronization pipeline should be resilient to bad data.

The goal is not to assume perfect source data, but to safely process valid information while isolating problematic records for later review.
