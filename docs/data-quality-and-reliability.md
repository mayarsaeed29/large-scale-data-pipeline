# Data Quality & Reliability

## Overview

This document describes the data-quality and reliability controls used in the large-scale data pipeline.

The goal of the pipeline was not only to process data successfully, but to ensure that incomplete, duplicated, corrupted, or incorrectly transformed data did not silently reach the analytical database.

Data quality is therefore treated as a separate responsibility within the workflow.

---

## Data Quality Objectives

The pipeline was designed to help ensure that processed data remained:

- Complete
- Consistent
- Traceable
- Free from confirmed duplicates
- Structurally valid
- Correctly transformed
- Suitable for analytical use

Validation is performed at multiple stages rather than only at the end of the pipeline.

---

## Multi-Stage Validation

Validation occurs across the workflow.

```text
Raw File
   ↓
Input Validation
   ↓
Structural Processing
   ↓
Transformation Validation
   ↓
Output Validation
   ↓
Database Publication
   ↓
Post-Publication Verification
```

This design reduces the risk of errors moving silently from one processing stage to another.

---

## Input Validation

Before a newly acquired file enters the processing workflow, basic checks are performed to confirm that it is suitable for ingestion.

Examples include:

- File existence
- Expected file type
- Expected naming structure
- Successful download completion
- Presence of required columns
- Basic structural integrity

Files that fail these checks are not treated as valid pipeline inputs.

---

## Schema Validation

Operational reports may change over time.

A missing, renamed, or unexpected column can break downstream transformations or produce incorrect analytical results.

The pipeline therefore checks the incoming schema before processing.

Typical checks include:

- Required columns are present
- Expected fields have not disappeared
- Unexpected structural changes are identified
- Data types can be interpreted as expected

Schema validation helps detect upstream changes before they affect reporting.

---

## Duplicate Protection

Duplicate prevention exists at multiple levels.

### File-Level Protection

Previously processed files should not be ingested again unintentionally.

The workflow tracks processing state and prevents repeated ingestion of the same source artifact.

### Record-Level Protection

Exact duplicate records are removed using controlled matching rules.

The purpose is to remove only confirmed duplicates while preserving legitimate records that may appear similar.

### Incremental Protection

New daily data is checked against the existing analytical dataset to reduce the risk of creating duplicate records during incremental loading.

---

## Why Exact Deduplication?

Operational datasets may contain records that look similar while representing different business events.

For this reason, the pipeline avoids aggressive fuzzy deduplication during the core processing workflow.

The principle is:

> Similar does not automatically mean duplicate.

Only records meeting the approved duplicate criteria are removed.

This helps preserve data integrity.

---

## Row-Count Validation

Row counts are monitored between important processing stages.

Unexpected changes can indicate problems such as:

- Excessive record removal
- Duplicate ingestion
- Parsing failures
- Filtering mistakes
- Transformation issues
- Partial input files

Row-count validation does not guarantee correctness by itself, but it provides a useful signal when combined with other checks.

---

## Missing-Value Validation

Missing values are not automatically considered errors.

Instead, they are evaluated based on the meaning of each field.

Depending on the column, a missing value may be:

- Expected
- Replaced using a controlled fallback
- Reconciled using another field
- Standardized to an approved representation
- Flagged for review

This prevents one generic missing-value rule from being applied to unrelated business fields.

---

## Transformation Validation

After business rules are applied, important transformations are checked to confirm that the resulting values are valid.

Examples include:

- Category normalization
- Mapping results
- Identifier reconciliation
- Fallback logic
- Derived values
- Standardized operational labels

Unexpected transformation results can cause the run to be flagged for review rather than silently published.

---

## Date Coverage Validation

Continuous reporting requires more than successful execution.

The analytical dataset must also contain the expected reporting periods.

The pipeline verifies represented dates to identify missing periods.

Conceptually:

```text
Expected Reporting Dates
          ↓
Compare With
          ↓
Dates Present in Analytical Data
          ↓
Identify Missing Periods
```

This allows data gaps to be detected even when the pipeline itself did not produce a technical error.

---

## Missed-Run Detection

Scheduled workflows can fail because of:

- Machine availability
- Authentication problems
- Network issues
- Reporting-system failures
- Download errors
- Scheduler configuration

The pipeline therefore does not assume that the previous scheduled execution succeeded.

Before acquiring new data, the controller determines which reporting periods are missing.

Any missing periods can then be recovered.

---

## Post-Ingestion Verification

Successful database insertion is not treated as the final proof that processing completed correctly.

After ingestion, the workflow verifies that the expected reporting period is represented in the analytical dataset.

This provides an additional control after database publication.

---

## File Publication Safety

Downloaded files are separated from approved pipeline inputs.

A file follows a controlled transition:

```text
Download
   ↓
Temporary Artifact
   ↓
Validation
   ↓
Approved File
   ↓
Pipeline Input
```

This helps prevent partially downloaded or invalid files from being processed automatically.

---

## Duplicate File Protection

Before publishing a newly acquired report, the workflow checks whether the same artifact has already been handled.

This helps protect against:

- Repeated downloads
- Scheduler retries
- Manual reruns
- Duplicate publication

Where appropriate, file metadata or content fingerprints can be used to distinguish already processed artifacts.

---

## Execution Locking

Only one active workflow should modify the same analytical pipeline at a time.

The system uses run locking to prevent overlapping executions.

```text
Workflow Starts
      ↓
Check Active Lock
     / \
   Yes  No
   ↓     ↓
 Stop  Continue
```

This protects the workflow from:

- Concurrent ingestion
- Duplicate processing
- Conflicting database writes
- Inconsistent state

---

## Controlled Execution Status

The pipeline distinguishes between different outcomes rather than using only success or failure.

Conceptually:

```text
PASS
REVIEW
FAILED
NO_NEW_DATA
```

### PASS

Processing and validation completed successfully.

### REVIEW

Processing completed, but one or more validation results require investigation.

### FAILED

A critical error prevented successful completion.

### NO_NEW_DATA

The workflow completed normally, but no new input required processing.

This provides more meaningful operational visibility than a simple binary status.

---

## Logging

Important stages produce logs that can be used to diagnose failures.

Examples of logged events include:

- Workflow start
- Reporting-period determination
- Authentication
- Report search
- Download initiation
- Download result
- File publication
- Ingestion
- Validation
- Database update
- Coverage verification
- Final execution status

This allows failures to be isolated to a specific stage.

---

## Failure Isolation

The staged architecture helps prevent a failure in one area from becoming difficult to diagnose.

For example:

```text
Acquisition succeeds
        ↓
File validation succeeds
        ↓
Transformation fails
```

The logs can show that acquisition and file validation were successful, narrowing the problem to the transformation stage.

This is more maintainable than treating the entire workflow as a single operation.

---

## Idempotency

The pipeline is designed so that rerunning an already processed period does not automatically create duplicate analytical records.

This is important because reruns may occur during:

- Failure recovery
- Manual investigation
- Scheduler retries
- Development and testing

Idempotent behavior makes recovery safer.

---

## Incremental Load Verification

New records are not assumed to be correct simply because they were inserted into PostgreSQL.

Incremental loads are checked to confirm that:

- The expected period was processed
- New data was actually added when appropriate
- No unintended duplicate records were introduced
- The analytical dataset remains consistent

---

## Rule Change Safety

Business rules may evolve over time.

Transformation changes are introduced in a controlled manner so that new logic does not silently alter previously validated behavior.

Rule changes should remain traceable through:

- Version control
- Clear documentation
- Explicit revisions
- Validation after changes

This helps distinguish data changes from logic changes.

---

## Reliability Principles

### Validate Early

Detect structural problems before expensive processing begins.

### Validate Again Before Publication

Transformation completion is not sufficient evidence that data is ready for analytics.

### Verify After Publication

Check that the expected data is actually represented after loading.

### Preserve Traceability

Maintain enough execution information to determine what happened during a run.

### Expect Failure

Scheduled automation should be designed with the assumption that some executions will eventually fail.

### Recover Safely

Missed periods should be detectable and recoverable without rebuilding the entire historical dataset.

### Avoid Silent Errors

Unexpected conditions should result in review or failure states rather than quietly producing questionable analytical data.

---

## Example Reliability Scenario

Consider a scheduled daily execution where report acquisition succeeds but the downloaded file is incomplete.

Without validation:

```text
Incomplete Download
        ↓
Pipeline Processes File
        ↓
Database Updated
        ↓
Reporting Uses Incomplete Data
```

With the reliability controls:

```text
Incomplete Download
        ↓
File Validation
        ↓
Validation Fails
        ↓
File Not Published
        ↓
Run Marked Failed / Review
        ↓
Issue Can Be Investigated
```

This illustrates why validation boundaries are important in automated data workflows.

---

## Data Quality as Part of the Architecture

Data quality was not treated as a final cleanup task.

It was integrated into the architecture itself.

The workflow combines:

**Validation → Controlled Processing → Verification → Traceability**

This approach helps make the analytical dataset more reliable for downstream reporting and analysis.

---

## Confidentiality

This document describes the quality-control approach used in the project without exposing:

- Production datasets
- Confidential business rules
- Internal identifiers
- Credentials
- Production database details
- Company systems
- Customer or passenger information

The examples focus on the engineering principles and reliability mechanisms behind the workflow.
