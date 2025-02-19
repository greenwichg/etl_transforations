<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" class="logo" width="120"/>

# Architecture for CSV-to-Parquet Data Pipeline in Log Processing

---

## Overview of Data Flow

The architecture consists of three primary zones: **Raw Zone**, **Aggregate Zone**, and **Operational Data Store (ODS)**. Below, we detail each component and its role in transforming raw CSV logs into optimized Parquet datasets.

---

## 1. **Raw Zone: Ingestion of Transactional Logs**

### Format and Structure

- **Data Format**: All incoming logs are stored as individual CSV files.
- **File Granularity**: Each transaction generates a separate CSV file (e.g., `txn_001.csv`, `txn_002.csv`).
- **Storage Location**: Files are stored in an S3 bucket or HDFS directory partitioned by date (e.g., `s3://raw-logs/2025-02-19/`).

**Example CSV Schema**:

```csv  
transaction_id,user_id,amount,timestamp  
TX001,USER123,150.00,2025-02-19T10:15:30Z  
```


### Challenges with Raw Data

1. **Small File Problem**: Thousands of CSV files degrade storage/query performance.
2. **Schema Variability**: Inconsistent column ordering or missing fields across CSVs.
3. **Uncompressed Storage**: CSVs lack compression, increasing storage costs.

---

## 2. **Processing Layer: Merging and Conversion**

### Step 1: Merging CSV Files

A Python script consolidates thousands of CSV files into larger chunks using **Pandas** or **Dask** for scalability:

```python  
import pandas as pd  
import os  

# Merge all CSVs in a directory  
def merge_csvs(input_dir, output_file):  
    csv_files = [f for f in os.listdir(input_dir) if f.endswith('.csv')]  
    dfs = [pd.read_csv(os.path.join(input_dir, f)) for f in csv_files]  
    merged_df = pd.concat(dfs, ignore_index=True)  
    merged_df.to_csv(output_file, index=False)  

merge_csvs('s3://raw-logs/2025-02-19', 'merged_2025-02-19.csv')  
```


### Step 2: Converting to Parquet

The merged CSV is converted to Parquet for optimized storage:

```python  
merged_df = pd.read_csv('merged_2025-02-19.csv')  
merged_df.to_parquet(  
    's3://aggregate-zone/2025-02-19/logs.parquet',  
    engine='pyarrow',  
    compression='snappy'  
)  
```

**Key Advantages of Parquet**:

- **Columnar Storage**: Faster analytics on specific columns (e.g., `amount`).
- **Compression**: Reduces storage costs by 60–80% compared to CSV.
- **Schema Evolution**: Supports adding/removing columns without breaking downstream processes.

---

## 3. **Aggregate Zone: Optimized Storage**

### Structure and Use Cases

- **Partitioning**: Data is partitioned by date/hour for time-range queries:

```  
s3://aggregate-zone/  
├── date=2025-02-19/  
│   └── logs.parquet  
```

- **Metadata Management**: Tools like **AWS Glue** catalog Parquet schemas for SQL queries.

**Performance Metrics**:


| Metric | CSV (Raw Zone) | Parquet (Aggregate Zone) |
| :-- | :-- | :-- |
| **Storage Size** | 1 TB | 200 GB (80% compression) |
| **Query Time** | 120 sec | 8 sec |

---

## 4. **ODS Integration: External Tables**

### Creating External Tables

Using **Redshift Spectrum** or **AWS Athena**, external tables map to Parquet files in the aggregate zone:

```sql  
CREATE EXTERNAL TABLE ods.transaction_logs (  
    transaction_id STRING,  
    user_id STRING,  
    amount DECIMAL(10,2),  
    timestamp TIMESTAMP  
)  
STORED AS PARQUET  
LOCATION 's3://aggregate-zone/';  
```


### Benefits for Consumers

1. **ACID Compliance**: Transactional consistency for concurrent access.
2. **Efficient Joins**: Columnar format accelerates joins with dimension tables.
3. **Cost Savings**: Reduced scan costs in query engines like Athena.

---

## Technical Challenges and Solutions

| **Challenge** | **Solution** |
| :-- | :-- |
| **Schema Drift** | Use AWS Glue crawlers to auto-detect schema changes in merged CSVs. |
| **Merge Failures** | Implement idempotent workflows with AWS Step Functions for retries. |
| **Small Files in Aggregate** | Use Dask/Spark to coalesce Parquet files into optimal sizes (128–256 MB). |

---

## Toolchain Recommendations

1. **Data Merging**:
    - **Pandas**: For small datasets (<10 GB).
    - **Dask/Spark**: For large-scale parallel processing.
2. **Conversion**:
    - **PyArrow**: For efficient Parquet writes with Snappy/Zstd compression.
3. **Orchestration**:
    - **AWS Step Functions**: To coordinate merging, conversion, and validation.

---

# Conclusion

By transitioning from fragmented CSV files in the raw zone to consolidated Parquet datasets in the aggregate zone, this architecture achieves:

- **Cost Efficiency**: Reduced storage/query costs via compression and columnar formats.
- **Performance**: Sub-second query response times for analytics.
- **Scalability**: Handles thousands of transactions daily without manual intervention.

For future enhancements, consider integrating **Delta Lake** for ACID transactions or **real-time processing** with Kafka for streaming use cases.

<div style="text-align: center">⁂</div>

[^1]: https://stackoverflow.com/questions/51696655/read-multiple-parquet-files-in-a-folder-and-write-to-single-csv-file-using-pytho

[^2]: https://www.tablab.app/csv/to/parquet

[^3]: https://mungingdata.com/python/writing-parquet-pandas-pyspark-koalas/

[^4]: https://www.youtube.com/watch?v=ja5MFgpdTAw

[^5]: https://codetinkering.com/combining-dataframes-python/

[^6]: https://docs.singlestore.com/cloud/load-data/load-data-with-pipelines/how-to-load-data-using-pipelines/create-a-parquet-pipeline/

[^7]: https://palantir.com/docs/foundry/building-pipelines/infer-schema/

[^8]: https://www.youtube.com/watch?v=aexszHMKdy8

[^9]: https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/three-aws-glue-etl-job-types-for-converting-data-to-apache-parquet.html

[^10]: https://mungingdata.com/dask/read-csv-to-parquet/

[^11]: https://dl.gi.de/bitstreams/9c8435ee-d478-4b0e-9e3f-94f39a9e7090/download

[^12]: https://stackoverflow.com/questions/74272379/how-to-sum-cell-data-from-multiple-csv-files-into-a-single-parquet

[^13]: https://community.databricks.com/t5/data-engineering/how-to-merge-small-parquet-files-into-a-single-parquet-file/td-p/11617

[^14]: https://www.youtube.com/watch?v=k280CpPKJgc

[^15]: https://palantir.com/docs/foundry/transforms-python/unstructured-files/

[^16]: https://www.upsolver.com/blog/apache-parquet-why-use

[^17]: https://duckdb.org/docs/data/parquet/overview.html

[^18]: https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-csv

[^19]: https://towardsdatascience.com/comparing-performance-of-big-data-file-formats-a-practical-guide-ef366561b7d2/

[^20]: https://www.reddit.com/r/Python/comments/13wp8bu/csv_or_parquet_file_format/

[^21]: https://community.databricks.com/t5/data-engineering/i-am-trying-to-read-csv-file-using-databricks-i-am-getting-error/td-p/12499

[^22]: https://aws.amazon.com/blogs/big-data/perform-upserts-in-a-data-lake-using-amazon-athena-and-apache-iceberg/

[^23]: https://community.fabric.microsoft.com/t5/Power-BI-Community-Blog/Parquet-ADLS-Gen2-ETL-and-Incremental-Refresh-in-one-Power-BI/ba-p/1706882

[^24]: https://learn.microsoft.com/en-us/fabric/onelake/onelake-medallion-lakehouse-architecture

[^25]: https://github.com/vdemichev/DiaNN

[^26]: https://discuss.python.org/t/how-to-convert-a-csv-file-to-parquet-without-rle-dictionary-encoding-error-message/18786

[^27]: https://www.confessionsofadataguy.com/converting-csvs-to-parquets-with-python-and-scala/

[^28]: https://github.com/BrunoArsioli/csv-to-parquet

[^29]: https://arrow.apache.org/docs/python/parquet.html

[^30]: https://gist.github.com/l1x/76dab6445b6d55396c622f915c755a17

[^31]: https://www.reddit.com/r/learnpython/comments/1f7rsjj/how_can_large_csv_files_in_python_be_efficiently/

[^32]: https://community.databricks.com/t5/machine-learning/merge-12-csv-files-in-databricks/td-p/3551

[^33]: https://duckdb.org/docs/data/multiple_files/overview.html

[^34]: https://lists.apache.org/thread/33rx1cgqymb8qkfo9trz5yzgdo51q17g

[^35]: https://towardsdatascience.com/hands-on-introduction-to-delta-lake-with-py-spark-b39460a4b1ae/

[^36]: https://aws.amazon.com/blogs/big-data/build-and-manage-your-modern-data-stack-using-dbt-and-aws-glue-through-dbt-glue-the-new-trusted-dbt-adapter/

[^37]: https://delta.io/blog/2023-03-22-convert-csv-to-delta-lake/

[^38]: https://docs.databricks.com/en/error-messages/error-classes.html

