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
- when invalid, store `null` and flag the record with:

`validationStatus = invalid_email`

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

`+55 (31) 99999-1001`

becomes:

`5531999991001`

---

### Status

Source values:

`active`

`inactive`

Target values:

`ACTIVE`

`INACTIVE`

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

```text
Batch with 100 records
        ↓
98 valid
2 invalid email
        ↓
98 processed normally
2 processed without email and flagged
