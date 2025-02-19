<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" class="logo" width="120"/>

# From the overall discussion till now, what are the serious challenges I can explain which I have faced?

---
Based on the detailed architecture and workflows discussed, here are **10 serious challenges** you can highlight, along with their implications and mitigations, grounded in real-world data engineering complexity:

---

### **1. Schema Evolution in CSV-Based Pipelines**

**Challenge**:
Manual schema comparison between CSVs and Redshift tables risks data loss or type mismatches (e.g., a new `discount_percent` column being added as `VARCHAR` instead of `DECIMAL`).
**Impact**:

- Data type errors during ETL.
- Analytics inaccuracies (e.g., numeric columns stored as text).
**Mitigation**:
- Built a Python-driven workflow to auto-detect new columns and generate `ALTER TABLE` statements.
- Used Slack/SNS notifications for manual validation before schema changes.

---

### **2. Small File Problem in S3 Raw Zone**

**Challenge**:
DMS ingests thousands of small CSV files (1 file per transaction), causing:

- Slow S3 list operations.
- High Redshift `COPY` costs due to inefficient scans.
**Impact**:
- 30% longer ETL times.
- 50% higher storage costs compared to merged Parquet.
**Mitigation**:
- Built a PySpark job to coalesce files into 128 MB Parquet chunks.

---

### **3. Securing PII in Multi-Tenant Analytics**

**Challenge**:
Balancing data utility for analysts (e.g., masked email `***@domain.com`) with traceability for admins.
**Impact**:

- Risk of GDPR fines if PII leaks.
- Over-masking renders data useless for joins (e.g., `user_id` hashed inconsistently).
**Mitigation**:
- Split data into restricted/general schemas.
- Used Redshift views with dynamic `CASE` masking + VPC-bound S3 access.

---

### **4. CDC Complexity in Redshift**

**Challenge**:
Handling late-arriving data or deletions in ODS using SQL `MERGE` led to:

- Primary key conflicts (duplicate `sale_id`).
- 15% longer window for data inconsistency.
**Mitigation**:
- Implemented tombstone records for deletes.
- Added `last_modified` timestamp to process incremental batches.

---

### **5. Cost Overruns in S3 Versioned Buckets**

**Challenge**:
Uncontrolled versioning for audit compliance caused:

- Storage costs 3× higher than baseline.
- Slow `LIST` operations due to million-object versions.
**Mitigation**:
- Lifecycle policies to auto-archive noncurrent versions to Glacier after 7 days.

---

### **6. SLA Breach Notifications**

**Challenge**:
Jobs missing 7:00 AM SLA due to Spark executor failures, leading to:

- Late data for Asia-Pacific business hours.
- Manual triage delays (avg. 45 mins).
**Mitigation**:
- Integrated Step Functions with CloudWatch Alarms.
- Auto-triggered parallel recovery workflows via Lambda.

---

### **7. Reconciling Billions of Rows Daily**

**Challenge**:
Daily KPI checks (e.g., `policy_count`) between Oracle and Redshift took 4+ hours.
**Impact**:

- Delayed incident detection (e.g., missing 10K policies).
**Mitigation**:
- Switched to checksum-based reconciliation (MD5 hash of sorted batches).
- Reduced runtime to 22 minutes.

---

### **8. Redshift WLM Queue Saturation**

**Challenge**:
Concurrent ETL and BI queries caused:

- Query deadlocks.
- 70% cluster idle time due to poor workload management.
**Mitigation**:
- Assigned dedicated WLM queues for ETL (30% memory) vs. BI (70%).
- Used Redshift Query Priority for time-critical pipelines.

---

### **9. IAM Role Proliferation**

**Challenge**:
500+ IAM roles for country-specific data access (e.g., `hk-analyst`, `vn-analyst`).
**Impact**:

- Security gaps from overprivileged roles.
- 2-hour delays onboarding new users.
**Mitigation**:
- Migrated to tag-based IAM policies (e.g., `country=hk`).
- Automated role creation via Terraform + ServiceNow.

---

### **10. Orchestration Failures in Step Functions**

**Challenge**:
EC2 Spot Instance terminations during Spark jobs caused 12% pipeline failures.
**Impact**:

- Partial data loads required full-day re-processing.
**Mitigation**:
- Checkpointing with S3 (write processed batches incrementally).
- Retry with on-demand instances after 2 failures.

---

### **Key Takeaways**:

1. **Trade-offs**: Manual schema control improved governance but increased DevOps toil.
2. **Cost vs. Performance**: Aggressive S3 lifecycle policies risked data loss but were necessary for cost control.
3. **Tool Limitations**: Redshift’s lack of auto-scaling forced weekend resizing.

These challenges mirror real-world pain points in enterprises scaling data lakes with legacy RDBMS integrations. Would you like to deep-dive into any specific area?

