# Retry and Error Handling Strategy

This document defines how the synchronization workflow should react to transient failures, invalid records and unexpected errors.

---

## Objectives

The error-handling strategy must:

- avoid stopping the entire synchronization because of a single bad record
- automatically retry temporary failures
- prevent infinite retry loops
- isolate records that require manual review
- preserve enough context for troubleshooting

---

## Error Categories

### 1. Validation Errors

Examples:

- invalid email
- invalid phone
- missing external ID
- invalid status

Expected behavior:

- handle the error at record level
- continue processing the remaining records
- attach a validation status
- route critical validation failures to an error flow

---

### 2. Transient API Errors

Examples:

- HTTP 429
- HTTP 500
- HTTP 502
- HTTP 503
- timeout
- temporary connection failure

Expected behavior:

- retry automatically
- use a limited retry count
- apply a delay between attempts
- stop retrying after the configured limit

Recommended strategy:

    Attempt 1
       ↓ failure
    Wait 5 seconds
       ↓
    Attempt 2
       ↓ failure
    Wait 15 seconds
       ↓
    Attempt 3
       ↓ failure
    Route to error handling

---

### 3. Permanent API Errors

Examples:

- HTTP 400
- HTTP 401
- HTTP 403
- invalid payload
- unsupported field

Expected behavior:

- do not retry indefinitely
- capture the response
- log the affected record
- route the item to the error handling workflow

Authentication errors should also trigger an operational alert.

---

## Retry Policy

Recommended maximum retry count:

`3 attempts`

Recommended delays:

- first retry: 5 seconds
- second retry: 15 seconds
- third retry: 30 seconds

This approach reduces unnecessary pressure on external services while still recovering from short-lived failures.

---

## Error Context

Every failed item should preserve enough information for investigation.

Recommended metadata:

    {
      "externalId": "CUS-1003",
      "workflowStep": "crm_upsert",
      "errorType": "HTTP_ERROR",
      "httpStatus": 503,
      "attempt": 3,
      "timestamp": "2026-08-14T14:00:00-03:00"
    }

Sensitive information must not be stored in logs.

---

## Dead-Letter Strategy

Records that cannot be processed after all retry attempts should be moved to a separate error queue or persistence layer.

Example:

    Main Workflow
          ↓
    Processing
          ↓
    Failure after retries
          ↓
    Dead Letter Queue
          ↓
    Manual review / reprocessing

The dead-letter record should contain:

- external ID
- original processing timestamp
- failed workflow step
- error category
- sanitized error message
- retry count

---

## Infinite Loop Prevention

The workflow must never retry a permanently invalid record indefinitely.

Each item should keep track of the number of attempts.

Example:

    retryCount >= 3
            ↓
    stop automatic processing
            ↓
    send to error queue

Records in the error queue should only return to the main workflow through an explicit reprocessing mechanism.

---

## Batch Resilience

A failure in one record must not automatically stop unrelated records.

Example:

    100 records
        ↓
    97 successful
    2 validation errors
    1 API failure
        ↓
    97 completed
    2 flagged
    1 sent to retry/error handling

---

## Observability

The workflow should track at least:

- total records received
- successful records
- validation failures
- API failures
- retries
- permanently failed records
- processing time

These metrics help detect systemic problems before they become operational incidents.

---

## Design Principle

Failures are expected events in distributed integrations.

The workflow should therefore be designed to recover automatically when possible and isolate failures safely when automatic recovery is not possible.
