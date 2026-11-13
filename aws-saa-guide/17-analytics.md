# 17 — Analytics

Lighter on the exam than the previous chapters, but "query data in S3 without
managing servers" is a guaranteed question.

## The services

| Service | What it is | Use |
|---------|------------|-----|
| **Athena** | **Serverless SQL over S3** | Ad-hoc queries, logs, data lake |
| **Glue** | Serverless **ETL** + **Data Catalog** | Crawl, transform, catalogue |
| **EMR** | Managed Hadoop / Spark / Hive / Presto | Big data processing at scale |
| **Redshift** | Data warehouse | BI, complex joins over TB–PB |
| **OpenSearch Service** | Search and log analytics | Full-text search, dashboards |
| **Kinesis Data Firehose** | Managed delivery stream | Land streaming data in S3 etc. |
| **QuickSight** | BI dashboards | Visualisation, ML insights |
| **Lake Formation** | Data lake governance | Fine-grained permissions on the lake |
| **MSK** | Managed Kafka | Kafka workloads |
| **Data Exchange** | Third-party datasets | Subscribe to external data |

## Athena

Serverless, pay **per TB scanned**, no infrastructure.

> **Exam tip.** "Query data in S3 using standard SQL with no servers to
> manage" → **Athena**. This phrasing is nearly verbatim on the exam.

Cost and performance levers — all four are examinable:

1. **Columnar formats** — **Parquet** or ORC instead of CSV/JSON
2. **Compression** — Snappy, gzip
3. **Partitioning** — by date/region so queries scan less
4. **Bucketing** — on high-cardinality columns

> **Exam tip.** "Reduce Athena cost" → convert to **Parquet** and
> **partition** the data. Both cut bytes scanned, which is what you pay for.

Athena uses the **Glue Data Catalog** for table definitions.

## Glue

| Component | Purpose |
|-----------|---------|
| **Data Catalog** | Central metadata store — used by Athena, EMR, Redshift Spectrum |
| **Crawlers** | Infer schema and partitions from S3 automatically |
| **ETL jobs** | Serverless Spark or Python shell |
| **Glue Studio** | Visual job authoring |
| **DataBrew** | Visual data preparation, no code |

> **Exam tip.** "Discover schema automatically and make S3 data queryable" →
> **Glue crawler** + **Data Catalog** + **Athena**.

## EMR vs Athena vs Redshift

| | Athena | EMR | Redshift |
|---|---|---|---|
| Model | Serverless SQL | Managed cluster | Provisioned (or serverless) warehouse |
| Data in | S3 | S3 / HDFS | Redshift storage (+ Spectrum for S3) |
| Best for | Ad-hoc, intermittent | Custom Spark/Hadoop, ML pipelines | Repeated complex BI queries |
| Pricing | Per TB scanned | Per instance-hour | Per node-hour |

> **Exam tip — the decision:**
> - Occasional queries over S3 → **Athena** (no idle cost)
> - Heavy, repeated BI with joins and concurrency → **Redshift**
> - Custom Spark/Hadoop frameworks, or lift-and-shift of existing Hadoop →
>   **EMR**

**Redshift Spectrum** queries S3 from Redshift, so hot data lives in the
warehouse and cold data stays in S3.

## Streaming analytics

```
Producers
   → Kinesis Data Streams  (replay, ordering, custom consumers)
   → Kinesis Data Firehose (managed delivery, no code)
       → S3 / Redshift / OpenSearch / Splunk
   → Managed Service for Apache Flink  (real-time SQL/Flink)
```

> **Exam tip.** "Stream logs into S3 with no code and near-real-time delivery"
> → **Firehose**. "Need replay or multiple independent consumers" →
> **Data Streams**. See [chapter 10](10-application-integration.md).

Firehose can **transform** records with Lambda and **convert to Parquet** in
flight — which directly feeds the Athena cost optimisation above.

## OpenSearch Service

Managed OpenSearch/Elasticsearch with Dashboards.

> **Exam tip.** "Full-text search", "centralised log search with dashboards",
> "near-real-time analytics on semi-structured data" → **OpenSearch**.

Common pipeline: CloudWatch Logs → Firehose → OpenSearch → Dashboards.

**OpenSearch Serverless** removes capacity management.

## QuickSight

Serverless BI. **SPICE** is its in-memory engine for fast dashboards.

> **Exam tip.** "Business dashboards for non-technical users, pay per session"
> → **QuickSight**. Row- and column-level security controls what each viewer
> sees.

## Lake Formation

Builds on Glue to govern a data lake: central permissions down to
table/column/row level, across Athena, Redshift Spectrum, EMR and QuickSight.

> **Exam tip.** "Grant analysts access to some columns but not others across
> the data lake" → **Lake Formation**. Managing that with bucket policies
> alone is the wrong answer.

## A reference data lake

```
Sources → Kinesis Firehose / DataSync / DMS
            ↓
          S3 (raw)  ──Glue ETL──►  S3 (curated, Parquet, partitioned)
            │                           │
      Glue Crawler → Data Catalog ──────┤
                                        ├─► Athena (ad-hoc SQL)
                                        ├─► Redshift Spectrum (BI)
                                        ├─► EMR (Spark / ML)
                                        └─► QuickSight (dashboards)
                   Lake Formation governs permissions throughout
```

> **Exam tip.** S3 is the storage layer of essentially every AWS data lake.
> An answer proposing a database as the lake is usually wrong.

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| SQL over S3, no servers | **Athena** |
| Reduce Athena cost | **Parquet** + **partitioning** |
| Discover schema in S3 automatically | **Glue crawler** |
| Serverless ETL | **Glue ETL** |
| Existing Spark/Hadoop jobs | **EMR** |
| Repeated complex BI queries | **Redshift** |
| Query S3 from the warehouse | **Redshift Spectrum** |
| Stream logs to S3, no code | **Kinesis Firehose** |
| Convert streaming data to Parquet | Firehose format conversion |
| Full-text search and log dashboards | **OpenSearch** |
| Dashboards for business users | **QuickSight** |
| Column-level permissions on a data lake | **Lake Formation** |
| Kafka workload | **MSK** |
| Real-time stream processing | **Managed Service for Apache Flink** |

## Check yourself

1. How is Athena priced, and what are the two main cost levers?
2. What does the Glue Data Catalog serve besides Glue?
3. Athena vs Redshift — what decides?
4. What does Redshift Spectrum let you avoid?
5. Firehose vs Data Streams for landing data in S3?
6. When is OpenSearch the right answer?
7. Why is Lake Formation better than bucket policies for a lake?
8. What is SPICE?

---

Previous: [16 — Migration and transfer](16-migration-and-transfer.md) ·
Next: [18 — Exam strategy](18-exam-strategy.md)
