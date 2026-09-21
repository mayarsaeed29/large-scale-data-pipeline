# Technical Design

## Overview

This document describes the technical design of the large-scale data pipeline documented in this repository.

The production system was designed to transform continuously arriving operational reports into validated, analysis-ready data stored in PostgreSQL.

The implementation follows a staged architecture so that ingestion, cleaning, transformation, validation, and publication remain separate and independently verifiable.

---

## Design Goals

The pipeline was designed around the following goals:

- Process large historical datasets reliably
- Support incremental daily ingestion
- Avoid unnecessary full-data rebuilds
- Prevent duplicate records
- Apply consistent transformation rules
- Detect incomplete or invalid data before publication
- Recover from missed scheduled runs
- Maintain traceability through execution logs
- Keep the analytical database synchronized with newly processed data
- Make individual pipeline stages easy to troubleshoot

---

## High-Level Architecture

```text
Operational Reports
        ↓
Acquisition Layer
        ↓
Raw Data
        ↓
Exact Deduplication
        ↓
Type Standardization
        ↓
Data Quality Review
        ↓
Missing-Value Processing
        ↓
Business Transformations
        ↓
Validation
        ↓
PostgreSQL
        ↓
Analytics / BI
```

The system separates data acquisition from transformation and database publication.

This prevents failures in one stage from silently affecting downstream analytical data.

---

## Historical Processing

The historical dataset is processed through a sequence of controlled stages.

Each stage produces an output that can be inspected and validated before progressing.

This staged approach provides several benefits:

- Easier troubleshooting
- Clear transformation boundaries
- Better auditability
- Reduced risk of silent data corruption
- Ability to rerun an individual processing stage when required

Historical processing establishes the validated analytical baseline used by downstream systems.

---

## Incremental Processing

Once the historical dataset has been established, normal operation focuses on newly arriving data.

The incremental workflow follows this pattern:

```text
Detect new reporting period
        ↓
Acquire new report
        ↓
Validate input file
        ↓
Process new records
        ↓
Apply transformation rules
        ↓
Validate processed output
        ↓
Publish to PostgreSQL
        ↓
Verify resulting coverage
```

This approach avoids repeatedly processing millions of previously validated records.

---

## Data Ingestion

The ingestion layer identifies newly available report files and determines whether they have already been processed.

Key responsibilities include:

- Detecting new files
- Preventing duplicate ingestion
- Preserving source files
- Recording processing state
- Passing approved files into the transformation workflow

Input data is treated as immutable source evidence wherever practical.

---

## Exact Deduplication

Deduplication is intentionally separated from broader cleaning logic.

Only records that satisfy the defined exact duplicate criteria are removed.

This avoids unintentionally merging records that appear similar but represent distinct operational events.

The principle is:

> Remove confirmed duplicates without changing legitimate business records.

---

## Type Standardization

Operational reports may contain inconsistent representations of:

- Dates
- Numbers
- Empty values
- Identifiers
- Categorical fields

The pipeline standardizes these fields before business transformations are performed.

This creates a predictable analytical schema and reduces downstream failures.

---

## Transformation Layer

Business transformations are applied only after the data has passed the early structural stages.

Transformation categories include:

- Value normalization
- Field reconciliation
- Category standardization
- Identifier cleanup
- Controlled fallback values
- Derived analytical fields

Transformation rules are managed separately from raw ingestion so changes in business logic do not require redesigning the entire pipeline.

---

## Rule Versioning

Operational mapping and transformation rules evolve over time.

Rather than silently overwriting previously validated logic, rule changes are introduced as controlled revisions.

This provides:

- Better traceability
- Safer incremental changes
- Easier debugging
- Clear understanding of when processing logic changed

---

## Validation Layer

Validation is treated as a separate system responsibility rather than being embedded only inside transformation code.

Examples of validation include:

- Required schema checks
- Row-count checks
- Duplicate checks
- Data-type checks
- Date coverage verification
- Transformation validation
- Incremental-load verification
- File integrity checks

A processing run is not considered successful simply because the transformation code completed.

The resulting data must also satisfy validation requirements.

---

## PostgreSQL Analytical Layer

Validated records are stored in PostgreSQL.

The database provides a centralized analytical layer between operational files and reporting tools.

Benefits include:

- SQL-based analysis
- Consistent reporting source
- Faster querying than repeatedly reading raw files
- Controlled schema
- Integration with BI tools
- Support for internal analytical applications

---

## Automated Acquisition

The daily workflow includes browser automation for acquiring new reports.

The acquisition component is responsible for:

- Authentication
- Navigation to the reporting interface
- Date selection
- Report generation
- Download initiation
- File verification
- Controlled publication

The acquired file is not immediately trusted.

It must first pass validation before becoming an approved pipeline input.

---

## Missed-Run Recovery

A scheduled pipeline cannot assume every previous execution succeeded.

The controller therefore evaluates which reporting periods are already represented in the analytical dataset.

If a gap is detected, the missing period can be acquired and processed before normal execution continues.

This creates a recovery-oriented design rather than relying on perfect scheduler execution.

---

## Run Locking

Only one pipeline execution should modify the analytical dataset at a time.

A non-blocking run lock prevents overlapping executions.

Conceptually:

```text
New execution starts
        ↓
Is another run active?
      /     \
    Yes      No
     ↓        ↓
Stop run   Continue processing
```

This helps protect against:

- Duplicate ingestion
- Conflicting database writes
- Simultaneous file processing
- Inconsistent execution state

---

## Publication Safety

The workflow separates downloaded artifacts from approved pipeline inputs.

```text
Downloaded Artifact
        ↓
Integrity / Validity Checks
        ↓
Approved Input
        ↓
Pipeline Processing
```

Files that do not satisfy expected requirements are not silently published into the processing area.

This reduces the risk of corrupted or incomplete downloads entering the analytical workflow.

---

## Scheduling

Normal execution is coordinated through scheduled automation.

The scheduled process is responsible for starting the workflow, while the pipeline itself remains responsible for determining what data actually needs to be processed.

This distinction is important:

```text
Scheduler
   ↓
"Start the pipeline"

Pipeline Controller
   ↓
"What reporting periods are missing?"
   ↓
"What needs to be acquired?"
   ↓
"What needs to be processed?"
```

The scheduler therefore does not contain the business logic for deciding data coverage.

---

## Logging

Important workflow events are logged so failures can be diagnosed.

Examples include:

- Acquisition start
- Authentication state
- Report request
- Download start
- Download result
- Ingestion start
- Validation result
- Database publication
- Coverage verification
- Final execution status

Structured logs make it possible to determine which stage failed rather than treating the entire workflow as a single black box.

---

## Failure Handling

The system distinguishes between different execution outcomes.

Conceptually, a run may finish as:

```text
PASS
REVIEW
FAILED
NO_NEW_DATA
```

This is more useful than treating every non-error execution as automatically successful.

A technically completed run may still require review if validation produces unexpected results.

---

## Why a Staged Architecture?

A single large script could technically perform many of these operations.

However, separating responsibilities provides better maintainability.

For example, if an error appears after a business-rule transformation, the previously standardized dataset can be inspected independently.

The pipeline therefore favors:

- Clear responsibilities
- Observable intermediate states
- Controlled transitions
- Validation boundaries

over one monolithic transformation process.

---

## Key Engineering Principles

### Reproducibility

The same input should produce the same transformation behavior under the same rule set.

### Idempotency

Reprocessing previously handled data should not create duplicate analytical records.

### Validation Before Publication

Successful execution does not automatically mean successful data publication.

### Incremental Processing

Daily operation should process only what is new whenever possible.

### Recoverability

The system should be able to identify and recover missing reporting periods.

### Traceability

Important processing and automation events should leave an observable record.

### Separation of Concerns

Acquisition, transformation, validation, and database publication should remain logically separate.

---

## Production Confidentiality

This document describes the architectural approach rather than the production implementation.

The following are intentionally excluded:

- Production source code
- Internal URLs
- Authentication details
- Credentials
- Proprietary mapping logic
- Confidential operational datasets
- Customer or passenger information

The purpose of this case study is to demonstrate the engineering decisions and problem-solving behind the system while preserving business confidentiality.
