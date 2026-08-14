# CRM Data Sync Platform

A production-oriented CRM data synchronization project designed to demonstrate how external systems can be integrated with a CRM through reliable automation workflows.

The project focuses on data ingestion, validation, normalization, pagination, retries, error handling and idempotent upserts.

---

## Overview

This project simulates a real-world integration between an external system and a CRM.

The main goals are:

- consume data from an external REST API
- process paginated responses
- validate and normalize records
- handle invalid data safely
- avoid duplicate processing
- retry transient failures
- perform idempotent upserts into the CRM
- maintain a clear and observable workflow

---

## Architecture

```text
External API
    ↓
n8n
    ↓
Pagination
    ↓
Validation & Normalization
    ↓
Error Handling / Retry
    ↓
Idempotent Upsert
    ↓
CRM
