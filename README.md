# Real-Time Data Ingestion & Transformation Pipeline  
*(AWS ➜ Snowflake via Airflow)*

> **Goal:** Deliver streaming CSV events from edge systems to Snowflake in under **1 minute**, using fully-managed, serverless components.

[🎬 **Watch the full implementation walkthrough on YouTube**](https://youtu.be/J0gI14uAmsA)

---

## Key Services

| Layer | Service | Purpose |
|-------|---------|---------|
| **Edge**        | EC2 (t2.micro) + Kinesis Agent | Stream synthetic `customers.csv` & `orders.csv` events |
| **Buffer**      | Kinesis Data Firehose          | Micro-batch (~5 MiB / 60 s) buffering → S3 landing      |
| **Lake**        | Amazon S3                     | Three-zone lake: `landing/` → `processing/` → `processed/` |
| **Orchestrator**| Amazon MWAA (Airflow 2.10)    | Detect new objects, run `COPY INTO`, trigger SQL transforms |
| **Warehouse**   | Snowflake                     | Raw & mart tables; external stage backed by S3           |

---

## Architecture

Every DAG run is parameterised with a UTC `BATCH_ID`, ensuring deterministic lineage and idempotent re-processing.  
Average end-to-end latency is **≈ 35 s** at **2 MB/s** throughput on a `t2.micro` plus X-SMALL Snowflake warehouse.  
See `docs/architecture.drawio` for the full diagram (exported above).

---

## Repository Layout
```text
.
├── dags/                         # Airflow DAGs
│   └── customer_orders_datapipeline_dynamic_batch_id.py
├── docker/                       # Local Airflow dev-container
├── terraform/                    # IaC for AWS + Snowflake objects
├── scripts/                      # Utility scripts (data generator, validation)
└── docs/
    ├── architecture.png
    └── presentation.pdf
```


---

## Deployment (AWS & Snowflake)

1. **Provision infrastructure**
   ```bash
   cd terraform
   terraform init && terraform apply
   ```
2. **Upload DAGs**
   ```bash
   aws s3 cp dags/ s3://$PIPELINE_BUCKET/dags/ --recursive
   ```
3. **Create Snowflake objects**
   ```bash
   snowsql -f scripts/snowflake_init.sql
   ```
4. **Start data generator**
   ```bash
   ssh ec2-user@<instance-ip>
   nohup python /opt/streamline/generate_data.py &
   ```

---

## Data Lineage & Idempotency

- **Immutable prefixes:** Files are copied to `processing/{BATCH_ID}/`, guaranteeing replayability.  
- **`BATCH_ID` column:** Stored in every raw & mart table; re-runs with the same ID generate **zero** new rows.

---

## Security Highlights

- Least-privilege IAM for EC2, Firehose, MWAA, and Snowflake storage integration  
- Credentials stored in **AWS Secrets Manager** and referenced via Airflow connections  
- End-to-end TLS across all services  

---

## Performance & Cost

| Metric               | Result                                    |
|----------------------|-------------------------------------------|
| End-to-end latency   | **35 s** mean (n = 20, 5 MiB batches)     |
| Throughput           | **2 MB/s** (EC2 network-bound)            |
| Snowflake warehouse  | CPU < 40 % (X-SMALL)                      |
| Cost (test volume)   | < $20 / month                              |

---

## Roadmap

- CI/CD for DAGs via AWS CodePipeline  
- MSK Connect for exactly-once semantics  
- dbt Core lineage inside Snowflake  
- Auto-schema detection with Snowflake  

---

## Authors

**StreamLine**  
Vishnu Vardhan Ciripuram (vc2499) · Vinay Varma Mudunuri (vm2622)

---

## License

MIT © 2025 StreamLine
