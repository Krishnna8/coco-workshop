---
name: prd-to-dt-plan
description: "Analyze PRD-style requirement files (XLSX, CSV) and produce a structured implementation plan for a target Dynamic Table. Use when: onboarding new sources, adding business rules, or updating a Silver/Gold layer DT from a PRD, BRD, or requirements spreadsheet. Triggers: prd, requirements to plan, onboard source, dynamic table plan, BRD analysis."
---

# PRD to Dynamic Table Plan

## When to Use

- A stakeholder delivers a PRD, BRD, or requirements spreadsheet describing new sources, field mappings, or business rules
- You need to translate those requirements into a concrete implementation plan for a Snowflake Dynamic Table
- You want a structured gap analysis before writing any SQL

## Inputs

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prd_path` | Yes | Path to the requirements file (XLSX or CSV). May be multiple files. |
| `target_dynamic_table` | Yes | Fully-qualified DT name (e.g., `ANALYTICS.SILVER.SILVER_AP_INVOICES`) |
| `existing_dt_sql` | No | Path to the current DT DDL or model file, if one exists. Used for gap analysis. |

## Workflow

### Step 1: Load and Parse the PRD

1. Read the file(s) at `prd_path`.
   - For XLSX: use the Read tool (supports `.xlsx`). Identify relevant sheets by name (look for keywords: "source", "mapping", "rules", "fields", "onboarding").
   - For CSV: read directly.
2. Classify each row/section into one of these categories:
   - **Source Onboarding** — new source systems, connectivity, refresh cadence
   - **Column Mapping** — source-to-target field mappings
   - **Business Rules** — transformations, normalization, deduplication, filtering
   - **Data Quality** — thresholds, alerts, validation checks
   - **Out of Scope** — items explicitly deferred or assigned to another layer/phase

**STOP**: Present the parsed summary to the user. Confirm nothing was misclassified before proceeding.

### Step 2: Identify Target DT Schema

1. If `existing_dt_sql` is provided, read it and extract the current column list, source UNION branches, and transformation logic.
2. If not provided, query the Snowflake catalog: `cortex search table-details "<target_dynamic_table>"` to get the current schema.
3. If the DT does not exist yet, note that this is a greenfield build and proceed to Step 3.

### Step 3: Gap Analysis

Compare PRD requirements against the current (or empty) DT definition. Produce:

1. **New columns** required but not present in the target DT
2. **New UNION branches** (one per new source system)
3. **Transformation changes** — new CASE statements, deduplication logic, type casts
4. **Upstream dependencies** — Bronze tables, staging tables, or connectors that must exist first
5. **Downstream impacts** — Gold tables or reports that consume this DT and may need updates

### Step 4: Surface Assumptions and Open Questions

This is the most critical step. For every requirement that is incomplete or ambiguous:

- **DO NOT guess or fill in defaults.** Instead, log it as an open question.
- Categorize each item:
  - **Blocking** — cannot implement without an answer (e.g., missing mapping for a required column)
  - **Non-blocking** — can implement a safe default but stakeholder should confirm (e.g., normalization layer choice)
- Include the original requirement ID/row reference so stakeholders can trace back.

**STOP**: Present open questions to the user. Do not proceed to the plan until blocking items are resolved or explicitly deferred.

### Step 5: Produce the Implementation Plan

Structure the plan as follows:

```
## Implementation Plan: <target_dynamic_table>

### Prerequisites
- [ ] Bronze tables / stages that must exist
- [ ] Permissions or legal approvals pending
- [ ] Reference tables needed (e.g., FX rates, CoA mapping)

### Schema Changes
| Column | Type | Source | Transformation | Rule Ref |
|--------|------|--------|----------------|----------|

### Source Branches (UNION ALL)
For each new source:
- Source system name and SOURCE_SYSTEM literal
- Bronze table reference
- Column mapping (source_col -> target_col)
- Source-specific transformations (status mapping, dedup, etc.)

### Business Rules Implementation
| Rule ID | Description | Implementation Approach | Layer |
|---------|-------------|------------------------|-------|

### Data Quality Checks
| Check | Type (DMF / assertion / alert) | Threshold | Rule Ref |
|-------|-------------------------------|-----------|----------|

### Out of Scope / Deferred
| Item | Reason | Target Phase |
|------|--------|-------------|

### Open Questions (Unresolved)
| # | Question | Blocking? | Owner | Original Ref |
|---|----------|-----------|-------|--------------|
```

### Step 6: Optional — Generate Skeleton SQL

If the user requests it, produce a skeleton DDL for the Dynamic Table incorporating the plan. Mark all assumption-dependent sections with `-- TODO: <open question ref>` comments.

## Stopping Points

- After Step 1: Confirm PRD parsing is correct
- After Step 4: Resolve blocking open questions before generating the plan

## Best Practices

### On Assumptions
- Never silently resolve ambiguity. If a rule says "normalize" but doesn't specify how, surface it.
- If two rules conflict, flag both and ask for priority.
- If a column exists in the PRD but has no source mapping, mark it blocking.

### On Scope
- Respect layer boundaries. If the PRD says "Gold layer," do not put it in the Silver plan.
- If a rule applies to multiple DTs, note it but only plan for the target DT.

### On Traceability
- Every planned change should reference back to a Rule ID, Request ID, or row number in the PRD.
- This makes stakeholder review possible without re-reading the whole PRD.

## Output

The skill produces a single structured implementation plan (markdown) containing:
1. Parsed PRD summary (categorized)
2. Gap analysis against current DT state
3. Ordered implementation steps with rule traceability
4. Open questions table with blocking/non-blocking classification
5. (Optional) Skeleton SQL

---

## Example Usage

**User prompt:**
```
/prd-to-dt-plan
prd_path: ./assets/sample_business_requirements_source_onboarding.xlsx
target_dynamic_table: ANALYTICS.SILVER.SILVER_AP_INVOICES
existing_dt_sql: ./models/silver/silver_ap_invoices.sql
```

**Expected output structure:**

```markdown
## Implementation Plan: ANALYTICS.SILVER.SILVER_AP_INVOICES

### Prerequisites
- [ ] Bronze table: RAW.BAAN.INVOICES (nightly CSV via S3)
- [ ] Bronze table: RAW.WORKDAY.INVOICES (hourly via connector INC-44291)
- [ ] Legal sign-off on DPA-2025-0041 (Workday)

### Schema Changes
| Column | Type | Source | Transformation | Rule Ref |
|--------|------|--------|----------------|----------|
| SOURCE_SYSTEM | VARCHAR | literal | Hard-coded per branch | BR-007 |
| INVOICE_STATUS | VARCHAR | varies | CASE normalization | BR-001 |

### Source Branches (UNION ALL)
**Baan (SRC-2025-003)**
- SOURCE_SYSTEM = 'BAAN'
- Bronze: RAW.BAAN.INVOICES
- Status: POSTED -> APPROVED (BR-001)
- Dedup: QUALIFY ROW_NUMBER() OVER (PARTITION BY INVOICE_NUMBER ORDER BY CREATED_AT DESC) = 1 (BR-003)
- Drop: BAN_COMPANY (BR-008)

**Workday (SRC-2025-004)**
- SOURCE_SYSTEM = 'WORKDAY'
- Bronze: RAW.WORKDAY.INVOICES
- Status: Approved -> APPROVED, In Review -> PENDING (BR-001)
- Drop: WD_TENANT_ID (BR-008)

### Open Questions (Unresolved)
| # | Question | Blocking? | Owner | Original Ref |
|---|----------|-----------|-------|--------------|
| 1 | Normalize payment terms at Silver or Gold? | Non-blocking | Sarah Chen / David Kim | BR-005 |
| 2 | Baan cost center format changed (BC-XX -> BC-XXX). Handle both? | Blocking | Karen van der Berg | SRC-2025-003 |
| 3 | BR-004 threshold is "$500K USD equivalent" but Silver has no FX conversion. How to evaluate for non-USD invoices? | Blocking | Tom Walsh | BR-004 |
```
