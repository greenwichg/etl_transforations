<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" class="logo" width="120"/>

# Comprehensive Explanation of Amazon Redshift Architecture for CDC and Data Warehousing

---

## Key Concepts in Amazon Redshift Data Warehousing

### 1. **Dimensions and Facts**

In a data warehouse, **dimensions** and **facts** form the backbone of a **star schema** or **snowflake schema**:

- **Dimensions** are descriptive attributes used to categorize and filter data (e.g., `Customer`, `Product`, `Time`).
    - Example (Sales Use Case):
        - `dim_customer`: Customer ID, name, location, demographics.
        - `dim_product`: Product ID, category, price, supplier.
        - `dim_time`: Date, month, quarter, fiscal year.
- **Facts** are numerical metrics representing business events (e.g., sales transactions).
    - Example:
        - `fact_sales`: Sales ID, customer ID (FK), product ID (FK), quantity sold, revenue, discount.

**Relationship**: Fact tables reference dimension tables via foreign keys to enable multi-dimensional analysis (e.g., "Total sales by region in Q1 2025")[^1][^2].

---

### 2. **ODS Layer (Operational Data Store)**

The **ODS layer** serves as a transient staging area that mirrors source systems. Its key roles:

- **Single Source of Truth**: Maintains an exact replica of raw data from transactional systems (e.g., Oracle) for traceability.
- **Data Harmonization**: Consolidates data from multiple sources (e.g., CRM, ERP) into a unified schema.
- **CDC Handling**: Captures incremental changes (inserts/updates/deletes) from source systems.

**Example Schema in ODS**:

```sql  
CREATE TABLE ods_sales (  
  sale_id INT,  
  customer_id INT,  
  product_id INT,  
  sale_date TIMESTAMP,  
  amount DECIMAL(10,2),  
  last_modified TIMESTAMP  -- Used for CDC  
);  
```

*Purpose*: The ODS layer acts as a buffer between raw S3 data and transformed warehouse tables, ensuring data integrity during ETL[^7][^9].

---

### 3. **Stored Procedures in Redshift**

Stored procedures automate complex workflows using procedural SQL (PL/pgSQL). Common uses:

- **CDC Processing**: Merge incremental data into target tables.
- **Data Quality Checks**: Validate referential integrity, null values, or outliers.
- **SCD Type 2**: Track historical changes in dimensions (e.g., customer address updates).

**Example CDC Stored Procedure**:

```sql  
CREATE PROCEDURE merge_incremental_sales()  
AS $$  
BEGIN  
  -- Merge data from ODS to fact_sales  
  MERGE INTO fact_sales AS target  
  USING ods_sales AS source  
  ON target.sale_id = source.sale_id  
  WHEN MATCHED THEN  
    UPDATE SET amount = source.amount  
  WHEN NOT MATCHED THEN  
    INSERT (sale_id, customer_id, product_id, sale_date, amount)  
    VALUES (source.sale_id, source.customer_id, source.product_id, source.sale_date, source.amount);  
END;  
$$ LANGUAGE plpgsql;  
```

*Benefits*: Redshift stored procedures reduce manual ETL effort and ensure transactional consistency[^20][^24].

---

### 4. **External Tables and Redshift Spectrum**

**Redshift Spectrum** allows querying data directly in Amazon S3 without loading it into Redshift clusters:

- **External Tables**: Metadata definitions pointing to S3 data (e.g., Parquet/CSV files).

```sql  
CREATE EXTERNAL TABLE spectrum_sales (  
  sale_id INT,  
  customer_id INT,  
  sale_date TIMESTAMP  
)  
STORED AS PARQUET  
LOCATION 's3://sales-bucket/raw/';  
```

- **Use Cases**:
    - **Cost Optimization**: Query historical data in S3 instead of storing it in Redshift.
    - **CDC from S3**: Join external tables with internal Redshift tables to process incremental updates[^12][^18].

---

### 5. **Change Data Capture (CDC) in Redshift**

CDC captures incremental changes from source systems (e.g., Oracle) and applies them to Redshift:

#### Methods for CDC:

1. **Timestamp-Based**:
    - Track `last_modified` columns in ODS tables.
    - Filter rows where `last_modified > MAX(last_run_timestamp)`.
2. **DMS (AWS Database Migration Service)**:
    - Continuously replicate Oracle changes to S3 via **Change Data Capture (CDC)**[^27][^28].
    - Redshift stored procedures process S3 files to update target tables.
3. **Merge Statements**:
    - Use `MERGE` to upsert data into fact/dimension tables[^24]:

```sql  
MERGE INTO fact_sales  
USING (SELECT * FROM ods_sales WHERE last_modified > '2025-02-18') AS source  
ON fact_sales.sale_id = source.sale_id  
WHEN MATCHED THEN UPDATE SET ...  
WHEN NOT MATCHED THEN INSERT ...;  
```


---

## Architecture for Sales Data Use Case

### End-to-End Flow

1. **Source System**:
    - **Oracle Database**: Hosts transactional sales data (e.g., `sales`, `customers`).
2. **Data Migration**:
    - **AWS DMS**: Replicates full data + CDC changes to S3 in Parquet format.
    - **S3 Raw Zone**: `s3://sales-bucket/raw/` stores daily snapshots.
3. **ODS Layer in Redshift**:
    - External tables (via Redshift Spectrum) map to S3 raw data.
    - Stored procedures clean/transform data into ODS tables.
4. **Data Warehousing**:
    - **Dimensions**: `dim_customer`, `dim_product`, `dim_time`.
    - **Facts**: `fact_sales`, `fact_inventory`.
    - **Transformations**: Aggregation, deduplication, SCD Type 2 logic.
5. **CDC Processing**:
    - Stored procedures run hourly to merge incremental changes from ODS to facts/dimensions.
    - Example quality checks:

```sql  
-- Check for orphaned customer IDs  
SELECT COUNT(*) FROM fact_sales  
WHERE customer_id NOT IN (SELECT customer_id FROM dim_customer);  
```

6. **Consumption Layer**:
    - Views/materialized views simplify querying for BI tools (e.g., Tableau).

---

### Key Components and Tools

| Component | Role | Example Tools/Features |
| :-- | :-- | :-- |
| **Source** | Transactional data generation | Oracle, MySQL, PostgreSQL |
| **CDC** | Capture incremental changes | AWS DMS, Debezium |
| **Storage** | Raw data lake | S3 (Parquet/CSV) |
| **ODS** | Staging area for raw data | Redshift external tables |
| **Transformations** | Data cleaning, SCD, aggregation | Redshift stored procedures, AWS Glue |
| **Data Warehouse** | Optimized for analytics | Redshift fact/dimension tables |
| **Orchestration** | Schedule ETL jobs | AWS Step Functions, Apache Airflow |

---

## Transformations Applied

1. **Data Cleaning**:
    - Remove duplicates using `ROW_NUMBER()`:

```sql  
DELETE FROM ods_sales  
WHERE ctid NOT IN (  
  SELECT MIN(ctid)  
  FROM ods_sales  
  GROUP BY sale_id  
);  
```

2. **Deduplication**:
    - Use `DISTINCT` or window functions to eliminate redundant rows.
3. **Slowly Changing Dimensions (SCD)**:
    - Track historical changes in `dim_customer`:

```sql  
CREATE TABLE dim_customer (  
  customer_key INT,  
  customer_id INT,  
  name VARCHAR,  
  address VARCHAR,  
  valid_from TIMESTAMP,  
  valid_to TIMESTAMP,  
  is_current BOOLEAN  
);  
```

4. **Aggregation**:
    - Precompute metrics like monthly sales:

```sql  
CREATE TABLE agg_monthly_sales AS  
SELECT  
  DATE_TRUNC('month', sale_date) AS month,  
  SUM(amount) AS total_sales  
FROM fact_sales  
GROUP BY 1;  
```


---

## Performance Optimization

- **Distribution Styles**: Use `KEY` distribution for fact tables to collocate joined data.
- **Sort Keys**: Sort `fact_sales` by `sale_date` for time-range queries.
- **WLM Queues**: Allocate memory for ETL vs. query workloads.
- **Redshift Spectrum**: Offload cold data to S3 to reduce cluster costs[^18][^12].

---

## Challenges and Solutions

| Challenge | Solution |
| :-- | :-- |
| **CDC Latency** | Use AWS DMS with parallel apply threads[^21][^27]. |
| **Data Consistency** | Wrap merges in transactions (`BEGIN`/`COMMIT`). |
| **Schema Drift** | Use AWS Glue crawlers to detect S3 file changes[^16]. |
| **Cost Management** | Archive old data to S3 and query via Spectrum. |

By integrating these components, your architecture ensures scalable, real-time analytics while maintaining data consistency across Oracle, S3, and Redshift[^10][^28].

<div style="text-align: center">⁂</div>

[^1]: https://en.wikipedia.org/wiki/Redshift

[^2]: https://lco.global/spacebook/light/redshift/

[^3]: https://aws.amazon.com/blogs/big-data/implement-a-slowly-changing-dimension-in-amazon-redshift/

[^4]: https://www.besanttechnologies.com/what-is-amazon-redshift

[^5]: https://www.sprinkledata.com/blogs/amazon-redshift-data-warehouse

[^6]: https://docs.aws.amazon.com/redshift/latest/dg/c_high_level_system_architecture.html

[^7]: https://stackoverflow.com/questions/63906276/creating-a-11-view-for-every-table-in-data-warehouse-ods-layer

[^8]: https://aws.amazon.com/redshift/modern-data-architecture/

[^9]: https://www.pingcap.com/case-study/why-we-chose-a-scale-out-data-warehouse-for-real-time-analytics/

[^10]: https://d1.awsstatic.com/Amazon_Redshift_Reference_Architectures_Powering_Customer_Success_V5_HQ_file.pdf

[^11]: https://www.amazon.science/latest-news/amazon-redshift-ten-years-of-continuous-reinvention

[^12]: https://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-external-tables.html

[^13]: https://www.youtube.com/watch?v=p09xfO8TtVo

[^14]: https://stackoverflow.com/questions/48338585/how-to-load-cdc-into-redshift-database

[^15]: https://aws.amazon.com/blogs/apn/change-data-capture-from-on-premises-sql-server-to-amazon-redshift-target/

[^16]: https://repost.aws/questions/QU9ykNVvWuQ1WRJ19SEr7f8w/redshift-data-warehouse-and-glue-etl-design-recommendations

[^17]: https://docs.matillion.com/metl/docs/2830845/

[^18]: https://www.integrate.io/blog/what-is-amazon-redshift-spectrum/

[^19]: https://bryteflow.com/cdc-to-amazon-redshift/

[^20]: https://aws.amazon.com/blogs/big-data/use-the-new-sql-commands-merge-and-qualify-to-implement-and-validate-change-data-capture-in-amazon-redshift/

[^21]: https://repost.aws/articles/ARlyMMJAeUQLWZd8p3c1Jz6A/automated-change-data-capture-cdc-data-ingestion-from-dynamodb-to-redshift

[^22]: https://stackoverflow.com/questions/48338585/how-to-load-cdc-into-redshift-database

[^23]: https://repost.aws/articles/ARMvA1y5XcT5KHJ2Jhnx27QQ/use-generic-logic-to-manage-data-warehouse-change-data-capture-cdc-in-amazon-redshift

[^24]: https://docs.aws.amazon.com/redshift/latest/dg/r_MERGE.html

[^25]: https://bryteflow.com/cdc-to-amazon-redshift/

[^26]: https://stackoverflow.com/questions/46604446/aws-dms-incremental-migration-from-oracle-to-redshift

[^27]: https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Target.Redshift.html

[^28]: https://repost.aws/articles/ARvAtOH0DMQVq9pCY7TPTG6w/amazon-redshift-database-as-a-target-for-aws-database-migration-service-dms-with-best-practices

[^29]: https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/migrate-an-oracle-database-to-amazon-redshift-using-aws-dms-and-aws-sct.html

[^30]: https://estuary.dev/oracle-to-redshift/

[^31]: https://aws.amazon.com/blogs/big-data/dimensional-modeling-in-amazon-redshift/

[^32]: https://stackoverflow.com/questions/50282011/redshift-performance-of-flat-tables-vs-dimension-and-facts

[^33]: https://www.logicmonitor.com/blog/what-is-amazon-redshift

[^34]: https://experiencestack.co/redshift-table-design-best-practices-3b7cb0cfccd6

[^35]: https://www.cloudzero.com/blog/aws-redshift/

[^36]: https://docs.hevodata.com/destinations/data-warehouses/amazon-redshift/redshift-data-structure/

[^37]: https://repost.aws/questions/QU4iHPD2_4TUKO5xjyKFSDAA/does-redshift-spectrum-support-incremental-ingestion

[^38]: https://docs.informatica.com/integration-cloud/data-integration-connectors/current-version/amazon-redshift-connectors/part-2--data-integration-with-amazon-redshift-v2-connector/mappings-and-mapping-tasks-with-amazon-redshift/before-you-begin/create-an-external-schema-and-table-for-amazon-redshift-spectrum.html

[^39]: https://docs.informatica.com/integration-cloud/data-integration-connectors/current-version/amazon-redshift-connectors/part-2--data-integration-with-amazon-redshift-v2-connector/mappings-and-mapping-tasks-with-amazon-redshift/amazon-redshift-v2-objects-in-mappings/sql-transformation/stored-procedure.html

[^40]: https://cloud.google.com/bigquery/docs/migration/redshift-sql

[^41]: https://bryteflow.com/oracle-cdc-change-data-capture-13-things-to-know/

[^42]: https://hevodata.com/learn/migrate-data-from-aws-rds-oracle-to-redshift/

