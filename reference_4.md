<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" class="logo" width="120"/>

# Comprehensive Framework for Diagnosing Data Mismatches and Failures in ETL Pipelines

---

## Systematic Approach to Data Discrepancy Resolution

When discrepancies arise in data pipelines, structured troubleshooting is critical to maintain data integrity and operational continuity. Below is a detailed methodology for addressing both **job failures** and **KPI mismatches**, leveraging AWS services like Step Functions, CloudWatch, and ECS.

---

## 1. **Job Failure Analysis**

### a. Step Function Execution Logs in CloudWatch

AWS Step Functions orchestrates workflows by coordinating AWS services (e.g., ECS tasks, Lambda). When a job fails:

1. **Review Execution History**:
    - Navigate to the Step Functions console → **Executions** tab → Failed execution → **Execution events**.
    - Identify the failed state (e.g., `LambdaFunctionFailed` or `TaskFailed`) and extract the `Cause` field containing error details[^1][^7].
2. **CloudWatch Log Groups**:
    - Step Functions automatically streams execution logs to CloudWatch if configured with `LogLevel: ALL`[^7].
    - Access logs via:

```bash  
/aws/vendedlogs/states/<StateMachineName>-Logs  
```

    - Filter logs using keywords like `ExecutionFailed` or `ERROR` to pinpoint failures[^4][^7].

### b. ECS Task Logs

For containerized tasks (e.g., Spark jobs):

1. **Cluster/Service Logs**:
    - In ECS → **Clusters** → Select cluster → **Tasks** → Failed task → **Logs**.
    - CloudWatch log streams are named:

```  
/ecs/<ClusterName>/<TaskID>  
```

2. **Common Errors**:
    - **Resource Exhaustion**: Check `MemoryUsage` or `CPUUtilization` metrics.
    - **Container Runtime Errors**: Inspect logs for Python/Java stack traces (e.g., `NullPointerException`).

### c. Remediation Strategies

- **Code Adjustments**:
    - Fix syntax errors or logic flaws (e.g., missing commas, incorrect API calls) identified in logs[^4].
- **Retry Mechanisms**:
    - Implement `Retry` policies in Step Functions for transient errors (e.g., throttling)[^7]:

```json  
"Retry": [  
  {  
    "ErrorEquals": ["States.ALL"],  
    "IntervalSeconds": 5,  
    "MaxAttempts": 3  
  }  
]  
```

- **Resource Scaling**:
    - Increase ECS task memory/CPU or optimize Spark partitions for large datasets.

---

## 2. **KPI Mismatch Investigation**

### a. KPI Validation Framework

KPIs like `policy_count` or `premium_amount` are validated through reconciliation SQL queries:

```sql  
-- Example: Premium Amount Reconciliation  
SELECT  
  SUM(source.premium) AS source_total,  
  SUM(target.premium) AS target_total,  
  ABS(SUM(source.premium) - SUM(target.premium)) AS discrepancy  
FROM oracle_policies AS source  
FULL OUTER JOIN redshift.fact_policies AS target  
  ON source.policy_id = target.policy_id  
WHERE discrepancy > 1000;  -- Tolerance threshold  
```

A discrepancy triggers alerts via SNS/SQS[^2][^5].

### b. Root Cause Analysis

1. **Trace Data Lineage**:
    - Identify the fact table (e.g., `fact_policies`) and trace dependencies:

```  
fact_policies → ods_policies → S3 (raw) → Oracle  
```

    - Use tools like AWS Glue DataBrew or OpenLineage to visualize transformations[^8][^11].
2. **Transformation Audits**:
    - Review stored procedures or Spark jobs responsible for:
        - Aggregations (e.g., `SUM(premium)`).
        - Joins (e.g., `policy_id` matching).
        - Data type conversions (e.g., `VARCHAR` → `DECIMAL`)[^6].
3. **Source-Target Comparison**:
    - Check for late-arriving data in S3 partitions or CDC gaps (e.g., missing DMS records)[^3].

### c. Corrective Actions

- **Data Backfilling**:
    - Reprocess affected partitions using historical S3 data.
    - Example for a date partition:

```sql  
ALTER TABLE spectrum_policies  
ADD PARTITION (date='2025-02-18')  
LOCATION 's3://policy-bucket/raw/date=2025-02-18/';  
```

- **Transformation Fixes**:
    - Adjust SQL logic (e.g., correct `GROUP BY` clauses).
    - Handle `NULL` values explicitly in ETL jobs[^3]:

```sql  
COALESCE(premium, 0) AS premium  
```


---

## 3. **Architecture for Failure Detection and Recovery

### a. Monitoring Stack

| Component | Role |
| :-- | :-- |
| **CloudWatch Metrics** | Tracks `ExecutionsFailed`, `ECSThrottledTasks`, and custom KPIs[^12]. |
| **CloudWatch Alarms** | Triggers SNS notifications for anomalies (e.g., `Discrepancy > 5%`) |
| **EventBridge Rules** | Routes failed executions to SQS DLQ for replay[^10]. |

### b. Automated Recovery Workflow

1. **Failure Detection**:
    - EventBridge rule triggers on `ExecutionStatus: FAILED`[^10].
2. **Error Classification**:
    - Lambda parses `Cause` from CloudWatch logs to categorize errors (e.g., `ResourceLimitExceeded`).
3. **Remediation**:
    - **Retry**: Replays failed executions with increased resources.
    - **Alert**: Sends Slack/email alerts via SNS for manual intervention.

---

## 4. **Preventive Measures**

### a. Data Quality Checks

- **Pre-Load Validation**:

```python  
# PySpark example  
df = spark.read.parquet("s3://raw-policies/")  
assert df.filter(df.premium < 0).count() == 0, "Negative premiums detected"  
```

- **Post-Load Reconciliation**:
    - Schedule AWS Glue DataBrew jobs to compare source/target counts hourly.


### b. Infrastructure Hardening

- **Step Functions Retry Policies**:

```json  
"Retry": [  
  {  
    "ErrorEquals": ["ECS.AmazonECSException"],  
    "IntervalSeconds": 10,  
    "BackoffRate": 2,  
    "MaxAttempts": 5  
  }  
]  
```

- **ECS Auto-Scaling**:
    - Configure Service Auto Scaling based on `CPUUtilization > 70%`.

---

## Conclusion

By integrating CloudWatch logging, Step Functions error handling, and data lineage tracing, teams can systematically resolve job failures and KPI mismatches. This approach not only accelerates root cause analysis but also minimizes data downtime through automated recovery workflows. For the described sales data architecture, implementing column-level lineage (e.g., via AWS Glue Data Catalog) would further enhance the ability to backtrack transformations for metrics like `premium_amount` or `policy_count`[^8][^11].

<div style="text-align: center">⁂</div>

[^1]: https://stackoverflow.com/questions/61727157/sending-exception-message-from-step-functions-to-aws-cloudwatch-event-logs

[^2]: https://helicaltech.com/10-key-performance-indicators-check-quality-data-etl/

[^3]: https://bigeval.com/dta/common-etl-data-quality-issues-and-how-to-fix-them/

[^4]: https://www.neenopal.com/cloud-watch-logs.html

[^5]: https://www.lightsondata.com/managing-dw-bi-data-integration-risks-through-data-reconciliation-and-data-lineage-processes/

[^6]: https://www.lonti.com/blog/handling-errors-maintaining-data-integrity-in-etl-processes

[^7]: https://docs.aws.amazon.com/step-functions/latest/dg/cw-logs.html

[^8]: https://www.secoda.co/blog/data-lineage-in-etl-process

[^9]: https://www.statsig.com/perspectives/etl-failures-common-fixes

[^10]: https://dev.to/thakurrishabh/aws-step-functions-workflow-for-an-etl-job-on-covid-19-and-deploying-it-with-terraform-cloudguruchallenge-series-part-2-3-6f7

[^11]: https://www.castordoc.com/blog/what-is-data-lineage

[^12]: https://docs.aws.amazon.com/step-functions/latest/dg/procedure-cw-metrics.html

[^13]: https://repost.aws/questions/QUVh12JYr-QXWL8OgOlHWDBQ/step-function-concurrent-job-is-failing-if-any-other-job-fails

[^14]: https://repost.aws/knowledge-center/glue-reduce-cloudwatch-logs

[^15]: https://github.com/aws-samples/aws-step-functions-etl-pipeline-pattern

[^16]: https://www.researchgate.net/publication/265254462_Quality_Measures_for_ETL_Processes

[^17]: https://www.linkedin.com/advice/0/how-do-you-measure-impact-value-data-lineage-tools

[^18]: https://www.montecarlodata.com/blog-5-etl-best-practices/

[^19]: https://hevodata.com/learn/etl-best-practices/

[^20]: https://buzzclan.com/data-engineering/what-is-etl/

[^21]: https://www.linkedin.com/advice/0/what-key-performance-indicators-etl-development-sklcf

[^22]: https://www.linkedin.com/advice/3/youre-facing-etl-process-issues-during-integration-agqvc

