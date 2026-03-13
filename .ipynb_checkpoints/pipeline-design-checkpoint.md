## Task 1.1 — End-to-End Pipeline Architecture

This pipeline is designed to process transactional data for an online retail company. The system must handle both historical batch data and new transactions arriving in real time. The goal of the pipeline is to ingest the data, validate it, transform it into analysis-ready form, and store it in structured layers so it can be used by analytics dashboards and machine learning models.

The architecture follows a layered design where raw data is first ingested and stored without modification. The data then passes through a validation stage where schema and business rules are checked. Invalid records are redirected to a quarantine area for investigation. Valid records continue to a transformation stage where additional features and derived columns are created. Finally, the processed data is stored in clean and feature layers that serve downstream consumers such as BI dashboards and ML models.

Below is the high-level architecture diagram of the pipeline.

                ┌──────────────────────┐
                │   Batch Data Source  │
                │  Online Retail.xlsx  │
                └───────────┬──────────┘
                            │
                            ▼
                ┌──────────────────────┐
                │    Batch Ingestion   │
                │   (Scheduled Load)   │
                └───────────┬──────────┘

 ┌──────────────────────┐   │
 │   Live Transaction   │───┘
 │       Stream         │
 │ (Real-time orders)   │
 └───────────┬──────────┘
             ▼
     ┌──────────────────────┐
     │     Ingestion Layer  │
     │  (Batch + Streaming) │
     └───────────┬──────────┘
                 ▼
        ┌───────────────────┐
        │     Raw Storage   │
        │    (Landing Zone) │
        │     Parquet       │
        └───────────┬───────┘
                    ▼
          ┌──────────────────┐
          │    Validation    │
          │ Schema & Rules   │
          └───────┬──────────┘
                  │
        ┌─────────▼─────────┐
        │ Quarantine / Dead │
        │    Letter Queue   │
        │ (Invalid Records) │
        └─────────┬─────────┘
                  ▼
         ┌──────────────────┐
         │   Transformation │
         │ Cleaning + Enrich│
         └─────────┬────────┘
                   ▼
         ┌──────────────────┐
         │   Clean Data     │
         │   Storage Layer  │
         └─────────┬────────┘
                   ▼
         ┌──────────────────┐
         │   Feature Layer  │
         │ Customer Features│
         └─────────┬────────┘
                   │
         ┌─────────▼──────────┐
         │      Consumers     │
         │                    │
         │  BI Dashboard     │
         │  ML Prediction    │
         └───────────────────┘

                 ▲
                 │
        ┌──────────────────┐
        │    Monitoring    │
        │ Freshness, Data  │
        │ Quality, Volume  │
        └──────────────────┘

In this pipeline, each stage has its own role. This pipeline helps us to process and validate raw data by batch and stream transaction from raw data, separate invalid valid data, transform, and finally save clean data. At the same time, it shows how BI and ML models access the data. Finally, it monitors our data.


# Task 1.2 — Component Descriptions

## Data Sources

The pipeline receives data from two sources. The first source is a historical dataset stored in the **Online Retail.xlsx** file. This dataset contains past transactions and is loaded once at the beginning. The second source is a live transaction stream that represents new orders arriving in real time. Both sources provide the raw transaction data used in the pipeline.

## Ingestion Layer

The ingestion layer collects data from both sources and sends it into the pipeline. For the batch source, a scheduled job reads the Excel file and loads the data. For the streaming source, transactions arrive one by one and are processed continuously. This layer makes sure the data is captured and passed to the next stage.

## Raw Storage (Landing Zone)

Raw storage is the first place where incoming data is saved. The data is stored exactly as it arrives, without cleaning or changes. This allows the pipeline to keep the original data in case it needs to be processed again later. The data is usually stored in **Parquet format** and organized by date to make processing easier.

## Validation Layer

The validation layer checks that each record follows the correct schema and rules. It verifies data types, required fields, and basic business constraints. For example, quantities should be valid numbers and prices should be reasonable values. Records that pass validation continue in the pipeline, while invalid records are sent to a quarantine area.

## Quarantine / Dead Letter Queue

The quarantine area stores records that fail validation. These records are separated from the main pipeline so they do not affect the processing of valid data. Engineers can review these records to understand what went wrong. This helps maintain data quality while allowing the pipeline to continue running.

## Transformation Stage

The transformation stage prepares the validated data for analysis. In this step, the pipeline cleans fields and calculates useful values such as **transaction revenue (Quantity × UnitPrice)**. It can also create aggregated information like the number of purchases per customer or total spending. These transformations make the data easier to analyze.

## Clean Data Layer

The clean data layer stores the processed and validated data. At this point the data is structured and ready for analysis. This layer is often used for reporting and analytics queries. Data is usually stored in **Parquet format** because it works well for analytical workloads.

## Feature Layer

The feature layer contains data prepared for machine learning models. It stores aggregated features such as total customer revenue, number of orders, and number of unique products purchased. These features are updated regularly so models always use recent information. This layer keeps ML features organized and separate from the clean data.

## Consumers

The pipeline provides data to two main consumers. One consumer is a **BI dashboard** that shows reports like daily sales, top products, and sales by country. The other consumer is a **machine learning model** that predicts whether a customer may become a high-value customer. Both systems use the processed data from the clean and feature layers.

## Monitoring and Observability

Monitoring helps ensure the pipeline runs correctly. The system tracks things like data freshness, data volume, and data quality. It can detect unusual changes such as missing data or sudden spikes in transactions. If a problem is detected, alerts can notify the data engineering team so they can investigate and fix the issue.



## Task 2.1 — Validation Rules

### Schema Validation (Structural Correctness)

| Field | Expected Type | Required | Validation Rule |
|------|---------------|----------|----------------|
| InvoiceNo | String | Yes | Must be a non-empty string |
| StockCode | String | Yes | Must not be null or empty |
| Description | String | No | Can be null but must be a string if present |
| Quantity | Integer | Yes | Must be a valid integer value |
| InvoiceDate | Datetime | Yes | Must be a valid timestamp |
| UnitPrice | Float | Yes | Must be a numeric value |
| CustomerID | Integer | No | If present, must be a valid numeric identifier |
| Country | String | Yes | Must not be empty |

---

### Value Range Validation (Sensible Values)

| Field | Rule | Description |
|------|------|-------------|
| Quantity | Quantity ≠ 0 | A transaction should not have zero quantity |
| UnitPrice | UnitPrice > 0 | Price must be positive for valid purchases |
| UnitPrice | UnitPrice < 10000 | Prevent unrealistic pricing values |
| InvoiceDate | InvoiceDate ≤ current date | Transaction date cannot be in the future |

---

### Business Rule Validation (Domain Logic)

| Rule | Description |
|-----|-------------|
| If InvoiceNo starts with "C", Quantity must be negative | Cancellation invoices represent returned items |
| If Quantity > 0, UnitPrice must be greater than 0 | Valid purchases must have a positive price |
| CustomerID with transactions must have a valid InvoiceNo | Every purchase must belong to a valid invoice |
| The same InvoiceNo should belong to a single transaction group | Prevent inconsistent invoice records |





## Task 2.2 — Error Handling Flow

### Schema Validation Errors

If a record fails schema validation (for example, a missing required field or incorrect data type), the record is rejected and sent to a **quarantine table**. This prevents invalid data from entering the main pipeline. The system also logs the error with details about the failed field. If many schema errors occur, the monitoring system sends an alert to the data engineering team so they can investigate the issue.

### Value Range Errors

If a record contains unrealistic values (for example, negative prices or zero quantity), the record is also sent to the **quarantine area**. These records are not processed further because they may represent incorrect data. Engineers can later review these records and determine whether the issue comes from the source system or from incorrect input data.

### Business Rule Errors

If a record breaks business rules (for example, a cancellation invoice with a positive quantity), the record is flagged and moved to the **dead letter queue**. These records require manual inspection because they may indicate a logical inconsistency in the data. Logging is also performed so that the issue can be traced later.

### Operator Alerts

The monitoring system tracks the number of validation failures in the pipeline. If the error rate becomes too high or unusual patterns appear, alerts are sent to the data engineering team. Alerts can be delivered through email, dashboards, or messaging systems so that the team can quickly identify the problem.

### Recovery of Quarantined Records

When the source issue is identified and fixed, quarantined records can be reviewed and corrected if necessary. After that, the records can be reprocessed through the validation stage. If the records pass validation, they are allowed to continue through the pipeline and enter the clean data layer.





## Task 3.1 — Transformations

| Transformation | Input | Output | Idempotent |
|---|---|---|---|
| Remove duplicate records | Validated transaction data | Unique transaction records | Yes |
| Handle missing descriptions | Description field | recover description using StockCode lookup or flag them for analysis| Yes |
| Standardize country names | Country field | Clean and consistent country values | Yes |
| Calculate line total | Quantity, UnitPrice | New column: `line_total = Quantity × UnitPrice` | Yes |
| Create cancellation flag | InvoiceNo | New column: `is_cancellation` (True if InvoiceNo starts with "C") | Yes |
| Extract date components | InvoiceDate | New columns: year, month, day | Yes |
| Calculate customer total revenue | Transaction data grouped by CustomerID | Customer feature: total_revenue | Yes |
| Calculate order count per customer | InvoiceNo grouped by CustomerID | Customer feature: order_count | Yes |
| Calculate product diversity | StockCode grouped by CustomerID | Customer feature: number of unique products purchased | Yes |
| Calculate customer recency | InvoiceDate grouped by CustomerID | Customer feature: days since last purchase | Yes |
| Create ML features from observation window | Customer transactions in observation period | ML features such as revenue, frequency, and diversity | Yes | 




## Task 3.2 — Storage Layers

| Layer | Contents | Format | Update Frequency | Retention |
|---|---|---|---|---|
| Raw | Original transaction data from batch file and live stream | Parquet | Batch load once + continuous stream updates | Long-term storage |
| Clean | Validated and transformed transaction data | Parquet | Updated daily | Medium-term retention |
| Feature | Aggregated customer features for machine learning | Parquet | Updated daily (features) and weekly (ML training) | Stored for model history |

### Raw Layer

The raw layer stores the original data exactly as it arrives from the sources. No cleaning or transformations are applied at this stage. This allows the pipeline to keep a full copy of the original records in case the data needs to be reprocessed later. Parquet is used because it is efficient for storing large datasets and works well with analytical tools.

### Clean Layer

The clean layer contains validated and transformed transaction data. At this stage, the records follow the correct schema and include derived fields such as line totals or cancellation flags. This layer is mainly used for analytics and reporting tasks. Parquet format is chosen because it supports fast queries and efficient storage.

### Feature Layer

The feature layer stores aggregated customer-level features that are used by machine learning models. Examples include total revenue, number of orders, product diversity, and recency. These features are updated regularly so the ML model can use recent data. Keeping features in a separate layer helps organize the pipeline and makes model training easier.




## Task 3.3 — Incremental Updates

### Tracking Processed Data

To track which data has already been processed, the pipeline uses a **high-water mark strategy**. This means the system stores the timestamp of the last processed transaction, such as the latest `InvoiceDate`. When new data arrives, the pipeline processes only records that have a timestamp greater than the last processed value.

The processed and cleaned data is stored in **Parquet files in the clean layer**. This allows the data to be reused later for analytics, reporting, and machine learning tasks. Because the data is stored in a structured format, it can be easily queried and analyzed over time.

### Handling Late-Arriving Records

Sometimes transactions may arrive later than expected due to delays in the source system or network issues. To handle this situation, the pipeline keeps a **small reprocessing window**. For example, the system may reprocess the last few hours or the previous day of data to ensure that late records are not missed.

Late-arriving records are still stored in the raw layer and can be included during the next batch processing step. This approach helps ensure that the dataset remains complete and accurate.

### Refreshing Customer-Level Features

Customer-level features are important for both analytics and machine learning models. These features include metrics such as total revenue, number of orders, product diversity, and recency.

The feature layer is typically **updated daily** so that dashboards and analytics tools always have recent data. Machine learning models may use these updated features for training or prediction tasks. In many cases, models are retrained **weekly** using the refreshed feature data.

This regular update process ensures that both business analytics and ML systems are using the most recent customer behavior data.