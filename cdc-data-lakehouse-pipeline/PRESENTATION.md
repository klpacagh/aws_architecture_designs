# CDC + Data Lakehouse Ingestion Pipeline
## Executive Presentation

---

Good morning, everyone. Thank you for your time today. I want to walk you through our proposed CDC and Data Lakehouse Ingestion Pipeline — a system designed to give every analyst, data scientist, and executive in this organization access to the freshest, most trusted version of our operational data, without ever touching our production systems.

I will cover the problem we are solving, how the architecture works, the specific design decisions and their trade-offs, our performance and cost profile, the failure scenarios we have engineered for, and a summary recommendation.

---

## The Problem We Are Solving

Today, our organization maintains customer, order, and inventory data across three to five isolated operational databases — a mix of PostgreSQL and Aurora MySQL systems. These are production OLTP systems, highly optimized for transactional workloads, not for analytics.

When our analysts want to answer business questions — which customers churned this week, how is inventory moving across regions, what is the current state of every open order — they either wait for an overnight batch job, run queries directly against production databases at the risk of degrading performance, or work with data that is hours or days old.

The business requirements are clear:

- **Freshness**: Every INSERT, UPDATE, and DELETE made in any of our operational databases should be queryable in our central analytical platform within **five minutes** of the source commit. Not overnight. Not after a Glue crawler completes. Within five minutes.
- **Correctness**: We need complete historical accuracy. When a customer's address changes, we need to know both the old address and when it changed. This is Slowly Changing Dimension Type 2 — a full audit history of every entity, not just its current state.
- **Governance**: Some of our data is sensitive — personally identifiable information, financial records. We need row-level and column-level access control enforced at query time, not by duplicating data into separate tables for each audience.
- **Scale**: We are handling approximately one hundred thousand change events per minute today, with burst capacity to three hundred thousand. Our source tables range from ten gigabytes to ten terabytes. We need a platform that handles both the smallest lookup table and our largest transaction history without re-architecting.
- **Durability and correctness under failure**: When a Spark job crashes mid-merge, when a network partition isolates our replication instance, when a DBA changes a column type without telling us — the data must be safe, the error must be detectable, and the recovery must be automatic or require minimal intervention.

---

## Architecture Overview

Our pipeline is organized into **seven layers**, each with a clearly defined responsibility:

1. **Source Systems and CDC Capture** — RDS and Aurora databases; AWS DMS captures every row-level change as a structured event
2. **Stream Ingestion and Schema Enforcement** — Amazon MSK (Managed Kafka) as the streaming backbone; Glue Schema Registry validates every record's structure before it enters the pipeline
3. **Stream Processing and Transformation** — EMR Serverless running Apache Spark Structured Streaming; applies SCD Type 2 logic, deduplication, and enrichment in near-real time
4. **Data Lakehouse Storage** — Amazon S3 with Apache Iceberg table format, organized in Bronze, Silver, and Gold medallion layers; ACID upserts via Iceberg's MERGE INTO operation
5. **Catalog and Governance** — Glue Data Catalog as the central metastore; AWS Lake Formation enforcing fine-grained row and column-level access control
6. **Query and Analytics** — Athena for serverless ad-hoc SQL, Redshift Spectrum for BI workloads, QuickSight for executive dashboards
7. **Monitoring and Data Quality** — CloudWatch for pipeline health metrics, Glue Data Quality for automated freshness and integrity checks

Data flows left to right through layers one through four — the ingestion path that must complete within five minutes. Layers five and six provide governed read access to all three Iceberg tiers. Layer seven operates as a cross-cutting observability plane across the entire system.

---

## Layer-by-Layer Walkthrough

### Layer 1: Source Systems and CDC Capture

Our change data capture engine is **AWS Database Migration Service**. DMS connects to the source PostgreSQL or Aurora MySQL databases via their native replication protocols — the PostgreSQL write-ahead log and the MySQL binlog — and converts every row-level change into a structured Avro event.

DMS operates in two modes that work together seamlessly. In **full-load mode**, DMS takes a consistent snapshot of each source table and writes the current state to our lakehouse — seeding the Bronze layer with the complete historical record. Once the full load completes, DMS records the exact log sequence number at which the snapshot was taken and transitions automatically to **CDC mode**, replaying every change from that point forward. There is no gap between the snapshot and the ongoing stream.

I want to address a question you may be forming: why DMS over a self-managed Debezium connector on MSK Connect? Debezium is excellent — it is arguably more powerful for Kafka-native architectures. But it requires a Kafka Connect cluster to manage, JVM heap tuning, connector restart orchestration, and operational expertise. DMS eliminates that entire tier. For teams that do not already run Kafka Connect infrastructure, DMS's managed operational model is a significant advantage. If we ever move to a Kafka-centric platform with existing Connect infrastructure, we can migrate.

### Layer 2: Stream Ingestion and Schema Enforcement

Every CDC event produced by DMS flows into **Amazon MSK** — our managed Kafka cluster. MSK provides the durable, ordered, replayable streaming backbone that decouples the CDC capture layer from the processing layer.

We have designed one Kafka topic per source table, with twelve partitions each, using the primary key hash as the partition key. This ensures that all changes to a given row land in the same partition, preserving per-row ordering — critical for SCD Type 2 correctness.

Before any event enters the Kafka topic, it must pass validation against **AWS Glue Schema Registry**. Every producer registers its Avro schema with the Registry and serializes records using the registered schema ID. If a DBA changes a column type in a source database in a way that breaks our schema contract — a non-null column becomes required, a string column becomes an integer — the Schema Registry rejects the new record and DMS routes it to a dead-letter topic. The change is quarantined immediately, an alarm fires, and the pipeline continues processing compliant records for all other tables.

The compatibility mode we use is **BACKWARD_TRANSITIVE**. This means that any consumer running any version of the schema can always read records produced by any older schema version. Additive changes — adding a new nullable column with a default value — are approved automatically. Breaking changes require explicit coordination and a schema version bump.

Why MSK over Amazon Kinesis Data Streams? The answer comes down to integration and ecosystem. AWS DMS has a native Kafka target endpoint — it writes directly to MSK topics out of the box. It has no native Kinesis target. More importantly, Kafka's consumer group model gives us independent fan-out: our Spark Structured Streaming job, a monitoring consumer, and any future real-time consumers can all read the same topics simultaneously without any additional configuration or cost per consumer. At the throughput levels we are operating — 1.7 megabytes per second — the operational overhead of MSK is justified.

### Layer 3: Stream Processing and Transformation

This is where raw CDC events are transformed into meaningful, historically accurate records in our lakehouse. The processing engine is **Apache Spark Structured Streaming**, running on **EMR Serverless**.

Spark reads from the MSK topic in micro-batches triggered every sixty seconds. Within each micro-batch, two operations occur:

**First**, raw Avro CDC events are written as-is to the Bronze Iceberg layer — an append-only, immutable record of everything that happened, exactly as it was received. Bronze is our audit trail and our recovery foundation.

**Second**, Spark applies SCD Type 2 logic to transform those CDC events into the Silver layer. For each arriving change event, Spark executes an Iceberg MERGE INTO operation: if a current record exists for that primary key, it closes it by setting `is_current = false` and `valid_to = commit_timestamp`. It then inserts the new record version as the current one, with `is_current = true` and `valid_from = commit_timestamp`. For DELETE events, Spark closes the current record without inserting a replacement — the row is logically deleted but its history is permanently preserved.

This MERGE INTO operation is **ACID**. Iceberg uses optimistic concurrency control: two Spark jobs writing to the same table will detect conflicts and one will retry. No partial commit is ever visible to readers. An Athena query running concurrently with a MERGE sees a consistent snapshot — either the pre-merge state or the post-merge state, never a mix.

Why EMR Serverless over AWS Glue Streaming? SCD Type 2 MERGE operations are memory-intensive: Spark needs to join the incoming CDC events against the current-version records in the existing Iceberg table to find rows to close. Glue Streaming's DPU-based pricing model bills by the hour at a minimum DPU count, which becomes expensive for the memory profile this join requires. EMR Serverless bills per vCPU-second, scales worker count automatically with the micro-batch workload, and gives us full control over Spark configuration — memory fractions, shuffle partition count, Iceberg catalog properties. For SCD-heavy workloads, the operational control and cost model of EMR Serverless is superior.

### Layer 4: Data Lakehouse Storage — Apache Iceberg

Our lakehouse is built on **Apache Iceberg** over **Amazon S3**, organized in three tiers.

**Bronze** is the raw CDC event layer — every change event exactly as received, partitioned by database name, table name, and event date. Bronze is append-only. Nothing in Bronze is ever modified.

**Silver** is the SCD Type 2 history layer — the authoritative record of every version of every business entity. If you want to know what a customer's email address was on a specific date three years ago, you query Silver. If you want the current state of all open orders, you query Silver with `WHERE is_current = true`.

**Gold** is the business aggregate layer — pre-joined, pre-aggregated tables optimized for BI consumption. Daily active users. Weekly revenue by region. Inventory position by SKU. Gold is refreshed from Silver on a fifteen-minute schedule.

The choice of **Apache Iceberg** as our table format is worth explaining carefully because it is central to the correctness guarantees of this system.

Iceberg provides ACID transactions — which means our MERGE INTO operations are atomic, consistent, isolated, and durable. It provides schema evolution without data rewrites — when we add a new column to a source table, we add it to the Iceberg table schema and existing data files simply return NULL for that column. It provides partition evolution — we can change how a table is partitioned as it grows without rewriting the existing data. And it provides time-travel — every version of every table is accessible via its snapshot history for the last thirty days, giving auditors and data scientists the ability to query historical states with a single SQL clause.

We chose Iceberg over Delta Lake because Iceberg has first-class native support in Athena, Glue, and EMR without requiring vendor-specific connectors or licenses. We chose it over Hudi because Iceberg's MERGE INTO semantics are more expressive for SCD Type 2 patterns, and its native AWS integration ecosystem is more mature today.

### Layer 5: Catalog and Governance

The **Glue Data Catalog** serves as our central metastore, storing table definitions, schema versions, and Iceberg metadata for every Bronze, Silver, and Gold table. Every query engine — Athena, EMR, Redshift Spectrum — reads table metadata from the Glue Catalog, ensuring a single source of truth for all schema definitions.

**AWS Lake Formation** is our governance layer. It operates as an interceptor at the Glue Catalog level: when any query engine requests data from a Lake Formation-governed table, Lake Formation evaluates the caller's identity and data filter rules before returning access credentials to the underlying S3 data.

This means that row-level and column-level security is enforced at query time — not at storage time, not by duplicating data. A data scientist querying the Silver customer table does not see the `ssn` or `credit_card_number` columns — Lake Formation removes them before the query reaches S3. A business analyst querying Gold aggregates sees exactly the pre-approved KPI columns and nothing else. An auditor with elevated permissions can query all columns and all rows.

Why Lake Formation over S3 bucket policies and IAM? Because S3 bucket policies operate at the object level — they cannot restrict which columns or rows within a Parquet file a user can read. Lake Formation provides column exclusion and row filtering at the catalog level, enforced consistently across Athena, Glue, and Redshift Spectrum, without requiring any application-level changes or data copies.

### Layer 6: Query and Analytics

Our query layer serves three distinct audiences with different latency and complexity requirements.

**Amazon Athena** serves data scientists and analysts who need ad-hoc SQL access. Athena is serverless — there is nothing to provision or manage. It has native Iceberg v2 support, meaning it can read our Silver and Gold tables, execute MERGE INTO (for manual corrections), and use time-travel syntax directly. Athena charges per terabyte of data scanned; Iceberg's metadata pruning and partition filtering significantly reduce the amount of data scanned per query, keeping costs low.

**Amazon Redshift Spectrum** serves our BI workloads — the dashboards and reports that run on schedules or respond to user interactions with sub-second latency. Redshift's MPP architecture and AQUA query accelerator are purpose-built for the aggregation-heavy queries that BI tools generate. Redshift Spectrum reads our Gold layer Iceberg tables directly from S3 via the Glue Catalog, benefiting from Iceberg's metadata pruning on top of Redshift's query planning.

**Amazon QuickSight** provides the executive dashboard layer. QuickSight connects to both Athena and Redshift as data sources, and its SPICE engine caches query results for sub-second dashboard refresh. Business users get self-service visualizations without needing SQL knowledge.

### Layer 7: Monitoring and Data Quality

The monitoring layer operates across every other layer in the pipeline. **CloudWatch** collects metrics from DMS, MSK, EMR Serverless, and Glue, and routes alarms to PagerDuty and Slack.

The six alarms we have defined cover the failure scenarios most likely to affect our freshness SLA or data correctness:

- DMS replication lag exceeding two minutes signals that our CDC capture is falling behind the source database.
- MSK consumer group lag exceeding ten thousand messages signals that our Spark processing has stalled.
- Silver freshness exceeding five minutes is a direct SLA breach alarm.
- Iceberg commit failure rate exceeding one percent signals concurrent write conflicts that may require job serialization.
- Glue Data Quality score below 0.95 triggers quarantine of failing rows and halts Gold layer promotion until the issue is resolved.
- Bronze-to-Silver row count drift exceeding five percent signals a potential data loss in the pipeline.

**Glue Data Quality** runs hourly against every Silver table and checks for freshness, null rates on non-nullable columns, uniqueness of primary keys in `is_current = true` records, and referential integrity against known lookup tables. Failing rows are quarantined to a `silver_quarantine` Iceberg table rather than blocking the pipeline, giving data engineers full visibility into the issue and a clean dataset to re-process after fixes.

---

## End-to-End Data Flow

Let me walk through exactly what happens when a single row changes in one of our production databases, with timing at each hop:

| Step | Component | Action | Cumulative Latency |
|------|-----------|--------|-------------------|
| 1 | Source database | Row committed (INSERT/UPDATE/DELETE) | 0 s |
| 2 | DMS replication instance | Reads WAL/binlog; serializes to Avro | +2–5 s |
| 3 | DMS → MSK | Event written to Kafka topic | +3–7 s |
| 4 | Glue Schema Registry | Schema validated inline | +0 s (< 10 ms) |
| 5 | EMR Serverless Spark | Micro-batch reads from MSK | +7–67 s |
| 6 | Spark → Bronze | Appends raw event to Bronze Iceberg | +72–82 s |
| 7 | Spark MERGE INTO | SCD Type 2 applied to Silver Iceberg | +82–112 s |
| 8 | Iceberg commit | Snapshot atomically visible to readers | +113–122 s |
| **Total** | **Source → Silver queryable** | | **~1.5 min (p50), ~4 min (p99)** |

At the median, a row change is reflected in the Silver layer in approximately **ninety seconds**. At the 99th percentile, approximately **four minutes**. Our SLA is five minutes — we have twenty percent headroom even at the 99th percentile.

This headroom matters. When a burst of bulk updates arrives — a mass account status change, a year-end inventory reconciliation — Spark's EMR Serverless auto-scaling absorbs it, the latency temporarily degrades, and then recovers. The five-minute SLA is designed with this burst behavior in mind.

---

## Key Performance Metrics

### Freshness Performance

| Layer | SLA | p50 Actual | p99 Actual | Buffer |
|-------|-----|-----------|-----------|--------|
| Bronze (raw events in S3) | < 2 min | ~1 min | ~1.5 min | 25% |
| Silver (SCD Type 2 complete) | < 5 min | ~2.5 min | ~4 min | 20% |
| Gold (aggregates) | < 15 min | ~10 min | ~13 min | 13% |

We meet all three freshness targets with meaningful margin at both the median and 99th percentile.

### Throughput Capacity

| Layer | Sustained Capacity | Burst Capacity | Scaling Mechanism |
|-------|-------------------|---------------|-------------------|
| DMS capture | 5,000 events/sec | 10,000 events/sec | Vertical scale replication instance |
| MSK ingest | 375 MB/s | 1 GB/s | Add brokers; increase partitions |
| EMR Serverless | 100K events/min | 300K events/min | Auto-scales worker count |
| Iceberg on S3 | Unbounded | Unbounded | S3 scales automatically |
| Athena queries | 20 concurrent | 100 concurrent | Workgroup limit increase |

### Availability

Our composite SLA across all serial ingestion path dependencies is approximately **99.6%**. With the key resilience design described in the failure modes section — events accumulate durably in MSK during Spark outages and are replayed idempotently — the effective data durability is **99.99%**. We do not lose data; we may temporarily lag in freshness during a recovery event.

---

## Cost Profile

At our sustained load of one hundred thousand CDC events per minute, our estimated monthly costs are:

| Service | Configuration | Monthly Cost |
|---------|---------------|-------------|
| AWS DMS | 2× r6i.xlarge replication instances | $500 |
| Amazon MSK | 3× kafka.m5.xlarge brokers, Multi-AZ | $900 |
| EMR Serverless | ~300 vCPU-hours/day (auto-scaled) | $600 |
| Amazon S3 | ~50 TB/month (Parquet + Iceberg metadata) | $1,200 |
| Glue ETL (compaction + maintenance) | Nightly jobs per table | $150 |
| Glue Data Quality | Hourly checks across 50 tables | $200 |
| Glue Schema Registry | 50 schemas, 500M serializations/month | $100 |
| Athena | ~100 TB scanned/month | $500 |
| Redshift Spectrum | 2× ra3.xlplus nodes | $700 |
| CloudWatch | Metrics, logs, alarms | $150 |
| **Total** | | **~$5,000/month** |

That is approximately **sixty thousand dollars per year** to provide near-real-time analytics over all our operational databases, with full historical SCD Type 2 records, fine-grained access control, and automated data quality enforcement.

There are four cost optimization levers we can apply as we mature the deployment:

**First**, S3 Intelligent-Tiering on the Bronze layer. Raw CDC events older than ninety days are accessed infrequently — primarily for compliance audits and backfill scenarios. Moving them to Infrequent Access tier automatically saves approximately forty percent on Bronze storage costs.

**Second**, Iceberg file compaction. Our streaming micro-batches produce many small files. Nightly compaction via Glue ETL merges them into larger optimal files, reducing both S3 GET request costs and the amount of data Athena scans per query — typically a thirty to fifty percent reduction in Athena costs.

**Third**, EMR Serverless pre-initialized workers during business hours. Pre-warming four worker nodes eliminates the thirty-to-sixty-second cold start latency during the day. Scaling to zero overnight eliminates cost during off-hours.

**Fourth**, MSK Tiered Storage for Kafka logs older than twenty-four hours. MSK Tiered Storage offloads older log segments to S3-backed storage at approximately fifty percent lower cost than broker-attached EBS, without any change to consumer behavior.

---

## Failure Modes and Resilience

I want to be direct about the failure scenarios we have engineered for, because resilience is not a feature you add later — it is designed in from the start.

### Scenario 1: DMS Replication Lag or Task Failure

DMS maintains a checkpoint — a log sequence number — that records its position in the source database's change log. If the DMS task crashes and restarts, it resumes from the last checkpoint. The source database binlog must be retained for at least twenty-four hours to ensure the checkpoint position is still valid. If the binlog has rotated past the checkpoint, DMS falls back to a full-load plus CDC restart — reseeding the Bronze layer from the current source snapshot. Because our Silver layer uses MERGE INTO, the re-inserted records are handled idempotently — no duplicates, no corruption.

The blast radius is temporary lag — the Silver layer becomes stale during recovery. No data is lost because MSK retains all events for seven days.

### Scenario 2: Schema Drift — Source Column Changes Break Avro Schema

This is the scenario that breaks pipelines silently at other organizations. When a DBA changes a column type or removes a column without notifying the data team, records produced under the new schema may corrupt the downstream tables if they are processed unchecked.

Our Schema Registry prevents this. The moment a record with a schema-incompatible structure arrives at DMS, the Registry rejects it, DMS routes it to the dead-letter topic, and a CloudWatch alarm fires. The affected table pauses updates; all other tables continue normally. The blast radius is isolated to the one table with the schema change, and the data engineer has a full queue of rejected events to replay after the schema is updated.

### Scenario 3: Bulk Update Burst Overwhelming MSK Throughput

When a batch update runs on a source database — mass status changes, inventory reconciliation — it can generate millions of CDC events in minutes. Our design absorbs this gracefully. MSK retains events for seven days regardless of consumption rate. EMR Serverless auto-scales worker count to process the backlog at maximum throughput. Freshness temporarily degrades during the burst and recovers as it subsides. No data is lost; no manual intervention is required.

### Scenario 4: Iceberg Commit Conflicts

When multiple Spark applications write to the same Iceberg table simultaneously, optimistic concurrency control may produce commit conflicts — one writer succeeds, the other detects the conflict and retries. Iceberg handles this automatically with exponential backoff and three retry attempts. In the rare case of persistent conflicts, we serialize jobs by introducing a dependency ordering between Spark applications writing to the same table. No data corruption is possible — Iceberg's ACID semantics ensure partial commits are never visible to readers.

### Scenario 5: Data Quality Regression

When a source application bug inserts records with NULL values in previously non-nullable columns — or when foreign keys reference deleted parent records — Glue Data Quality detects the violation on its next hourly run. Failing rows are quarantined to a separate Iceberg table rather than being rejected or silently ingested into Silver. The data engineering team is alerted, investigates the quarantine table, and either re-processes the quarantined rows after fixing the source issue or adjusts the quality rule if the schema has legitimately changed. Gold layer promotions that depend on the affected Silver table are paused until quality is restored.

---

## Data Consistency Model

I want to be transparent about the consistency trade-offs in this system.

Our pipeline operates under **at-least-once delivery** from DMS: in failure and retry scenarios, a CDC event may be delivered to MSK more than once. We mitigate this with two mechanisms. Within Spark, offset checkpointing ensures exactly-once processing within a single micro-batch. Across batches, the Bronze layer deduplication key `(primary_key, commit_lsn)` prevents duplicate events from being re-processed into Silver.

Iceberg's snapshot isolation means that Athena queries always see a consistent snapshot — never a partial MERGE. A query that starts while a MERGE is in progress will see either the pre-merge or post-merge state, never a mix of both.

Our SCD Type 2 history is permanent. We never delete records from the Silver layer, even for entities that have been deleted from the source. The deletion is recorded as a record close — `is_current = false`, `valid_to = commit_timestamp` — and the full history is always available for compliance and audit.

Time-travel is available for the last thirty days via Iceberg's snapshot history. For compliance scenarios requiring access to states older than thirty days, the raw Bronze layer retains the complete event history for five years.

---

## Summary and Recommendation

To summarize the architecture we are proposing:

A **seven-layer pipeline** that captures every change from our operational databases and delivers it to a governed, analytically optimized data lakehouse within five minutes, with full SCD Type 2 historical accuracy and zero data loss.

**AWS DMS** handles capture without any code changes to source applications — it reads directly from the database replication log. **Amazon MSK** provides the durable, replayable streaming backbone with built-in fan-out for multiple consumers. **Glue Schema Registry** enforces schema contracts at the point of production, preventing schema drift from corrupting downstream tables. **Apache Iceberg on S3** provides the ACID transactional semantics required for correctness under CDC upserts, schema evolution, and SCD Type 2 history — capabilities that simple Parquet append patterns cannot provide. **AWS Lake Formation** enforces row and column-level security consistently across all query engines without data duplication.

The total cost is approximately **five thousand dollars per month** — a fraction of the engineering time previously spent maintaining manual extract jobs and resolving data inconsistencies.

Every failure mode has a documented detection mechanism, mitigation strategy, and recovery path. The most critical property of this architecture is its **correctness under failure**: when things go wrong — and in distributed systems, things go wrong — the system accumulates events durably in MSK, recovers its processing position from Kafka offsets, and replays events idempotently through Iceberg's MERGE INTO. The result is a Silver layer that is eventually correct, always queryable, and never silently corrupted.

This design replaces a fragile collection of overnight batch jobs with a resilient, near-real-time data platform that will serve the organization's analytical needs for years. I am confident it is the right architecture for where we need to go.

I welcome your questions.

---

*Architecture diagram: see `cdc_data_lakehouse_pipeline.png` in the pipeline directory.*
