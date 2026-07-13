# Azure Data Factory Healthcare Claims Pipeline

## Project Overview

This project is an end-to-end healthcare claims processing pipeline built using **Azure Data Factory** and **Azure Data Lake Storage Gen2**. It ingests clinic claim files, validates provider NPIs using the **CMS NPI Registry API**, masks sensitive patient data, applies data-quality rules, enriches claims using ICD-10 reference data, and generates reporting-ready summaries.

## Business Problem

Healthcare claim files may contain invalid provider NPIs, exposed SSNs, rejected claims, negative billed amounts, missing allowed amounts, and unmatched diagnosis codes. The goal of this project is to automate the validation, cleaning, enrichment, and summarization of these files before they are used for reporting.

## Architecture

```text
ADLS Gen2 Source Claims
        |
        v
Claims Ingestion Pipeline
        |
        v
ADLS Gen2 Staging
        |
        v
NPI Validation Pipeline
        |
        v
CMS NPI Registry API
        |
        v
Claims Transformation Data Flow
        |
        v
ADLS Gen2 Reporting Layer
```

The ICD-10 reference file is copied from GitHub into the staging layer and joined with the claims data during transformation.

## Main Pipelines

### ClaimsIngestionPipeline

Reads files from `source/claims/`, checks whether they match the expected `Claims_Clinic*.csv` naming pattern, and copies valid files into `staging/claims/`.

Activities used:

* Get Metadata
* ForEach
* If Condition
* Copy Activity

### ClaimsNPIValidationPipeline

Validates provider NPIs through the CMS NPI Registry API using a reusable pipeline parameter called `p_provider_npi`.

```text
https://npiregistry.cms.hhs.gov/api/?number=<NPI>&version=2.1
```

An NPI is considered valid when the API returns `result_count > 0`.

### ClaimsNPIValidationDriverPipeline

Reads provider NPIs from staged claim files and passes them into the reusable NPI validation pipeline using Lookup and Execute Pipeline activities.

### ClaimsValidationTransformPipeline

Runs an ADF Mapping Data Flow that:

* Filters rejected claims
* Removes records with negative billed amounts
* Masks SSNs
* Fills missing allowed amounts
* Joins ICD-10 reference data
* Flags valid and invalid diagnosis codes
* Aggregates claim totals by provider and payer
* Writes reporting-ready output

### MasterPipeline

Orchestrates the complete workflow:

```text
ClaimsIngestionPipeline
        |
        v
ClaimsNPIValidationDriverPipeline
        |
        v
ClaimsValidationTransformPipeline
```

## Data Flow Logic

```text
Source Claims
    |
    v
Select Columns
    |
    v
Filter Invalid Records
    |
    v
Mask SSN and Clean Amounts
    |
    v
Join ICD-10 Reference
    |
    v
Aggregate Claims
    |
    v
Reporting Sink
```

Important transformations include:

```text
Filter negative and rejected claims:
toDecimal(billed_amount) >= 0 && claim_status != 'REJECTED'
```

```text
Mask SSN:
concat('XXX-XX-', right(toString(ssn), 4))
```

```text
Fill missing allowed amount:
billed_amount * 0.8
```

Final output is stored in:

```text
reporting/claims_summary/
```

## Technologies Used

* Azure Data Factory
* Azure Data Lake Storage Gen2
* ADF Mapping Data Flow
* CMS NPI Registry REST API
* GitHub
* CSV
* Dynamic ADF expressions

## Key Skills Demonstrated

* Dynamic file ingestion
* Parameterized datasets and pipelines
* REST API integration
* Get Metadata and ForEach processing
* Reusable child pipelines
* PHI and PII masking
* Data-quality validation
* Reference-data joins
* Aggregation
* Pipeline orchestration
* ADF troubleshooting
* GitHub source control

## Repository Structure

```text
adf-healthcare-claims-pipeline/
│
├── README.md
├── data/
│   ├── claims/
│   └── reference/
├── screenshots/
├── pipeline/
├── dataset/
├── dataflow/
├── linkedService/
└── trigger/
```

## Security

This repository contains only synthetic healthcare data. No real patient data, SSNs, credentials, storage keys, connection strings, or production PHI/PII are included.

## Project Outcome

The completed pipeline converts raw healthcare claim files into validated, cleaned, enriched, and aggregated datasets that can be used safely for reporting and analytics.

## Future Improvements

* Validate every unique NPI in each file
* Add rejected-record logging
* Integrate Azure Key Vault
* Add failure notifications
* Add incremental processing
* Create a Power BI dashboard
* Add CI/CD deployment
