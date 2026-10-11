<div align="center">

# Telecom ETL Pipeline using SSIS

**A file-driven ETL workflow for telecom transaction processing, data transformation, validation, and controlled loading into SQL Server.**

</div>

---

## Overview

This project implements a telecom transaction ETL workflow using **SQL Server Integration Services (SSIS)** and **Microsoft SQL Server**. It is based on a business scenario in which a telecom system periodically produces CSV files containing customer activity records.

The SSIS package iterates through files in a configured source directory, applies the transformation and lookup steps defined in the workflow, loads transaction data into SQL Server, routes source and destination errors to separate tables, and archives files after successful processing.

The implementation recreates the core ETL concept of the reference task using SSIS rather than Scala.

## Business Requirements

The pipeline is designed around the following requirements:

- Process multiple CSV files from a configured folder.
- Apply the specified transaction-field mappings and derived-column transformations.
- Resolve subscriber identifiers using IMSI reference data.
- Load processed transactions into a SQL Server destination table.
- Route source-side and destination-side errors to separate database tables.
- Archive files after successful processing.

## Technology Stack

| Technology | Role |
|---|---|
| SQL Server Integration Services (SSIS) | ETL orchestration and data-flow execution |
| Microsoft SQL Server | Relational storage for reference, transaction, and error-output tables |
| CSV flat files | Source data format |
| SSIS Variables and Foreach Loop Container | File iteration and dynamic file processing |
| Lookup Transformation | Subscriber ID lookup using IMSI reference data |
| Derived Column Transformation | Field derivation and data preparation |
| OLE DB Destination | Loading processed data and routed error outputs into SQL Server |

## ETL Architecture

The workflow is organized into a Control Flow layer and a Data Flow layer.

### Control Flow

The Control Flow uses a **Foreach Loop Container** to iterate over CSV files in the configured folder. Variables are used to support file-by-file processing. For each file, the package executes the ETL Data Flow and proceeds to the archive step after successful processing.

<!-- Add the Control Flow screenshot at docs/images/control-flow.png -->
<p align="center">
  <img src="docs/images/control-flow.png" alt="SSIS Control Flow" width="900">
</p>

### Data Flow

The Data Flow reads transaction records from a CSV flat file, applies derived-column logic, looks up subscriber information, derives the subscriber ID, calculates TAC and SNR values, and loads the resulting records into the transaction table. Source-side and destination-side error outputs are routed to separate database tables.

<!-- Add the Data Flow screenshot at docs/images/data-flow.png -->
<p align="center">
  <img src="docs/images/data-flow.png" alt="SSIS Data Flow" width="900">
</p>

## Source Data Specification

The source CSV contains the following transaction fields, as specified in the task brief.

| Column | Data Type | Length | Nullable | Sample Value |
|---|---|---:|:---:|---|
| `ID` | Integer | — | No | `156` |
| `IMSI` | String | 9 | No | `310120265` |
| `IMEI` | String | 14* | Yes | `490154203237518` |
| `CELL` | Integer | — | No | `123` |
| `LAC` | Integer | — | No | `12` |
| `EVENT_TYPE` | String | 1 | Yes | `1` |
| `EVENT_TS` | Timestamp | — | No | `16/1/2020 7:45:43` |

\*The task specification lists IMEI length as 14, while the sample value contains 15 digits and the mapping rules specify a length check of 15. Use the length rule configured in the actual package.

## Transformation and Mapping Rules

The following table documents the target mapping requirements described in the task specification. Confirm that each rule matches the final SSIS package configuration before presenting it as implemented behavior.

| Source Field | Mapping / Business Rule | Target Field |
|---|---|---|
| `ID` | Pass through without transformation | `Transaction_id` |
| `IMSI` | Pass through; reject the record if null | `IMSI` |
| `IMSI` | Look up the subscriber using IMSI reference data; assign `-99999` when no subscriber ID is available | `subscriber_id` |
| `IMEI` | Extract the first 8 characters; assign `-99999` if null or shorter than 15 characters | `TAC` |
| `IMEI` | Extract the last 6 characters; assign `-99999` if null or shorter than 15 characters | `SNR` |
| `IMEI` | Pass through without transformation | `IMEI` |
| `CELL` | Pass through; reject the record if null | `CELL` |
| `LAC` | Pass through; reject the record if null | `LAC` |
| `EVENT_TYPE` | Pass through without transformation | `EVENT_TYPE` |
| `EVENT_TS` | Validate the timestamp; reject the record if null or invalid | `EVENT_TS` |

## Database Design

The project uses the following tables in the `SSIS_Telecom_DB` database.

| Table | Responsibility |
|---|---|
| `dim_imsi_reference` | Reference dataset used to resolve subscriber identifiers from IMSI values. |
| `fact_transaction` | Destination table for processed telecom transaction records. |
| `err_source_output` | Stores records or errors routed from the source-side error output. |
| `error_destination_output` | Stores errors routed from the destination-side error output. |

## Error Handling

The package separates source-side and destination-side error outputs to support troubleshooting and operational review.

- **Source-side errors:** routed to `err_source_output`.
- **Destination-side errors:** routed to `error_destination_output`.

The specific errors captured depend on the SSIS error-output settings and the configured data conversions. Business-rule rejection behavior should be documented according to the validation logic implemented in the package.

## File Processing and Archiving

The package processes CSV files from a configured source folder using a `Foreach Loop Container` and SSIS variables. After a file's Data Flow completes successfully, the package executes the archive step to move or copy the processed file to the configured archive location, according to the task configuration.

## Results

Use this section to show the records loaded into `fact_transaction` after a successful package execution. Add a screenshot captured from SQL Server Management Studio (SSMS).

```sql
USE SSIS_Telecom_DB;
GO

SELECT TOP (100) *
FROM dbo.fact_transaction;
```

<!-- Add the result screenshot at docs/images/loaded-data.png -->
<p align="center">
  <img src="docs/images/loaded-data.png" alt="Processed transaction records in SQL Server" width="900">
</p>

*Example query for inspecting the loaded transaction records. Replace the screenshot placeholder with results from your own execution.*

## Repository Structure

Suggested repository layout:

```text
Telecom-ETL-Pipeline-SSIS/
├── docs/
│   └── images/
│       ├── control-flow.png
│       ├── data-flow.png
│       └── loaded-data.png
├── SSIS project files
├── SQL scripts
└── README.md
```

Adjust this structure to reflect the files actually included in the repository.

## Setup and Execution

1. Open the SSIS solution in a compatible Visual Studio / SQL Server Data Tools environment.
2. Create or restore the `SSIS_Telecom_DB` database and required tables.
3. Configure the OLE DB connection manager for the target SQL Server instance.
4. Configure the source and archive folder paths and the package variables used for file iteration.
5. Verify the CSV connection manager and column metadata against the input file format.
6. Execute the package.
7. Validate the loaded records in `fact_transaction` and review `err_source_output` and `error_destination_output` as applicable.

Exact setup steps may vary based on the SQL Server, SSIS, and Visual Studio versions used.

## Implementation Notes

- This repository documents an SSIS implementation of the ETL scenario described in the task brief; the reference learning series implements the concept using Scala.
- The mapping table summarizes the specified requirements. Ensure the final documentation matches the transformations and validation rules actually configured in the package.
- Do not commit credentials, private connection strings, or sensitive customer data.
- Replace all screenshot placeholders with screenshots from the project before publishing.
