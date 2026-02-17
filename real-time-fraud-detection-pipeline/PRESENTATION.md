# Real-Time Fraud Detection Pipeline
## Executive Presentation

---

Good morning, everyone. Thank you for taking the time to join this session. Today I want to walk you through our proposed Real-Time Fraud Detection Pipeline — the architecture, the economics, the risk posture, and the operational model that will underpin our transaction security going forward.

I want to be thorough here because this system sits directly in the critical path of every financial transaction we process. If we get this right, we protect our customers and our bottom line. If we get it wrong, we either miss fraud or we slow down legitimate commerce. So let us get into it.

---

## The Problem We Are Solving

Today, our organization processes approximately **400 million transactions per day**. That is roughly **5,000 transactions per second** on average, with peak bursts that can reach **25,000 transactions per second** during high-volume periods such as holidays, flash sales, and end-of-month payroll cycles.

Every single one of those transactions needs to be evaluated for fraud **before** it is authorized. Not after. Not in a batch job that runs overnight. Before the customer sees a response on their screen.

The business requirements are non-negotiable:

- **Latency**: We must return a fraud decision in under **200 milliseconds** end-to-end. Our target is a median of **50 milliseconds** and a 99th percentile of **100 milliseconds**. Anything slower degrades the customer experience and risks cart abandonment.
- **Accuracy**: Our model must maintain a **precision above 95%** and a **recall above 90%**. In plain terms, when we flag a transaction as fraud, we need to be right at least 95% of the time. And we need to catch at least 90% of actual fraud attempts.
- **Availability**: The scoring path must be available **99.9% of the time**. When it is not, we fall back gracefully to a rule-based engine — we never leave the door wide open.
- **Durability**: We cannot lose a single transaction record. Regulatory compliance demands a complete, immutable audit trail with a **seven-year retention** policy.
- **Continuous improvement**: The model must evolve. Fraud patterns shift constantly. We need an automated retraining pipeline that can deploy an improved model **within one hour** of training completion.

---

## Architecture Overview

Our pipeline is organized into **seven distinct layers**, each with a clearly defined responsibility. I want to walk through them one by one, because understanding the separation of concerns is critical to understanding why this design is resilient, cost-effective, and maintainable.

The layers are:

1. **Ingestion** — Accept and buffer incoming transactions
2. **Feature Store** — Enrich each transaction with real-time and historical signals
3. **Real-Time Inference** — Score the transaction against our deployed ML model
4. **Post-Processing and Actions** — Route the decision: approve, decline, or send to manual review
5. **Data Lake and Compliance** — Archive everything for audit, analytics, and retraining
6. **Model Training** — The offline loop that continuously improves our detection capability
7. **Monitoring and Observability** — End-to-end visibility into system health and model performance

Data flows left to right through layers one through four — that is the **hot path**, the real-time scoring path that must complete in under 200 milliseconds. In parallel, a **warm path** branches from layer one into layer five, writing every transaction to our data lake asynchronously. And a **cold path** feeds back from the data lake into layer six, where labeled data is used to retrain and improve our models.

This separation means that the real-time scoring path is never burdened by analytics workloads, and our data science team can iterate on models without touching the production inference stack.

---

## Layer-by-Layer Walkthrough

### Layer 1: Ingestion

Transactions enter the system through **Amazon API Gateway** over HTTPS. API Gateway handles TLS termination, request validation, throttling, and API key authentication. It supports **10,000 requests per second** per region with burst capacity to 5,000 concurrent connections.

From API Gateway, each transaction is written to **Amazon Kinesis Data Streams**, which serves as our streaming backbone. Kinesis provides ordered, durable, and replayable message delivery.

We have sized Kinesis at **20 shards**. Each shard handles 1 megabyte per second of write throughput, giving us a total capacity of **20 megabytes per second** — sufficient for 10,000 transactions per second at our average payload size of 2 kilobytes. We use **enhanced fan-out** so that each downstream consumer gets dedicated read throughput without contention.

A natural question is: why Kinesis over Amazon Managed Streaming for Apache Kafka? The answer is operational simplicity. At our throughput level of 5,000 to 10,000 TPS with fewer than five consumers, Kinesis is simpler to operate and costs roughly **$800 per month** compared to MSK's minimum of approximately **$1,500 per month** for a three-broker cluster. If we ever scale beyond 50,000 TPS, MSK becomes the better choice, but that is not where we are today.

### Layer 2: Feature Store

This is where each raw transaction is transformed into a rich feature vector that the model can score against. The feature engineering process runs on **AWS Lambda**, triggered directly by Kinesis via event-source mapping. We run one Lambda invocation per shard, giving us 20 concurrent processors.

The feature engineer performs the following steps for every transaction:

1. Reads **rolling aggregates** and **device fingerprints** from **Amazon ElastiCache for Redis**. This is a six-node cluster of r6g.xlarge instances running in cluster mode, capable of **200,000 operations per second** with sub-millisecond read latency.
2. On a cache miss, it falls back to **Amazon DynamoDB**, which provides single-digit millisecond reads. DynamoDB runs in on-demand capacity mode to handle spikes without capacity planning, and has point-in-time recovery enabled for compliance.
3. Computes **derived features** — transaction velocity, geographic distance between consecutive transactions, merchant risk scores, and similar signals.
4. Writes **updated rolling aggregates** back to Redis so the next transaction from the same customer benefits from the freshest data.
5. Passes the enriched feature vector — approximately **200 features, 1 kilobyte serialized** — to the inference orchestrator.

Why Lambda here and not ECS or Fargate? Because this workload is stateless and bursty. Lambda's per-invocation billing and automatic scaling from Kinesis shard count is a natural fit. An ECS service would require us to provision and pay for capacity even during low-traffic periods.

We also considered **SageMaker Feature Store** as an alternative to our Redis-plus-DynamoDB approach. The deciding factor was latency. SageMaker Feature Store's online store delivers reads in 10 to 50 milliseconds. Our Redis layer delivers them in under **1 millisecond**. When your total latency budget for the entire hot path is 100 milliseconds at the 99th percentile, every millisecond counts.

### Layer 3: Real-Time Inference

The inference layer is where the enriched feature vector meets the ML model. An **orchestrator Lambda** invokes a **SageMaker Real-Time Inference** endpoint, retrieves the fraud score, applies business rules on top of the raw model output, and publishes the decision.

The SageMaker endpoint hosts our model — currently an XGBoost ensemble, approximately **50 megabytes** in size. The endpoint auto-scales between **2 and 20 instances** of ml.c5.xlarge, using target-tracking on the `InvocationsPerInstance` metric.

Let me break down the inference latency budget:

| Hop | Median (p50) | 99th Percentile (p99) |
|-----|-------------|----------------------|
| Lambda invocation overhead | 1 ms | 3 ms |
| Network transit to SageMaker | 1 ms | 2 ms |
| Model inference (XGBoost) | 2 ms | 5 ms |
| Business rule evaluation | 1 ms | 2 ms |
| **Subtotal** | **5 ms** | **12 ms** |

That is 12 milliseconds at the 99th percentile for the inference stage alone. Well within our budget.

SageMaker also gives us capabilities that are critical for safe model deployment: **A/B testing** through production variant routing, **shadow mode** where a new model scores traffic without affecting decisions, and **automatic rollback** if performance degrades during a deployment.

Why SageMaker over hosting the model directly in Lambda? Lambda imposes a 10-gigabyte memory ceiling and lacks native ML deployment tooling — no model versioning, no A/B testing, no auto-scaling on inference-specific metrics. SageMaker is purpose-built for this.

### Layer 4: Post-Processing and Actions

Once a transaction is scored, the decision must be routed to the appropriate downstream action. We use **Amazon EventBridge** as the decision router because it supports content-based routing on score thresholds, decoupling the scoring logic from the action logic entirely.

The decision thresholds — which are configurable via **AWS Systems Manager Parameter Store** and can be adjusted without a deployment — are as follows:

| Score Range | Action |
|-------------|--------|
| 0.0 to 0.3 | **Approve automatically** |
| 0.3 to 0.7 | **Route to manual review** via Amazon SQS |
| 0.7 to 1.0 | **Decline and alert** via Amazon SNS |

For declined transactions, **SNS** fans out notifications to multiple subscribers: email, SMS, PagerDuty, and Slack webhooks, ensuring fraud analysts are alerted immediately.

For manual review cases, transactions are enqueued in **SQS** with a dead-letter queue for failure handling. An **AWS Step Functions** workflow then orchestrates the investigation process: assigning an analyst, collecting evidence, recording the decision, and — critically — feeding the analyst's label back into our training data for model improvement.

### Layer 5: Data Lake and Compliance

Every transaction — whether approved, declined, or sent for review — is archived in our data lake. This happens asynchronously through the warm path, so it adds zero latency to the scoring decision.

**Amazon Kinesis Data Firehose** consumes from our Kinesis stream via enhanced fan-out and delivers records to **Amazon S3**. Firehose handles batching, Snappy compression, and conversion to **Parquet** format automatically. Records are partitioned by date and hour for efficient querying.

On the storage side, we apply **lifecycle policies**: data stays in S3 Standard for 90 days, transitions to Infrequent Access for cost savings, and moves to Glacier after one year. All objects are protected by **S3 Object Lock** in WORM (Write Once Read Many) compliance mode, ensuring immutability for regulatory purposes. Versioning is enabled, and all access is logged via CloudTrail.

**AWS Glue Crawlers** run hourly to discover schema changes and update the **Glue Data Catalog**, which serves as our central schema registry. **Amazon Athena** provides serverless SQL access over the data lake, used by fraud analysts for investigation and by data scientists for exploratory analysis.

### Layer 6: Model Training Pipeline

Our model is not static. Fraud evolves, and our detection capability must evolve with it. The training pipeline operates on two cadences:

- **Weekly full retrain**: The model is retrained from scratch on the complete labeled dataset.
- **Daily incremental update**: The model is fine-tuned on the most recent labeled transactions.

The pipeline is orchestrated by **AWS Step Functions** and proceeds as follows:

1. **AWS Glue ETL** joins transaction data from S3 with analyst labels, producing a balanced training dataset. This Spark-based job handles deduplication, feature engineering for offline features, and class balancing.
2. The training dataset is versioned in **S3** for full reproducibility.
3. **SageMaker Training Jobs** execute the model training, using **spot instances** for a **70% cost reduction** on compute. Automatic hyperparameter tuning optimizes model performance.
4. **SageMaker Ground Truth** provides human-in-the-loop labeling for ambiguous cases, with active learning to prioritize the most informative samples for labeling.
5. Data scientists use **SageMaker Notebooks** for prototyping and validation.

Deployment follows a **champion/challenger** model. The new model is initially deployed to serve **5% of traffic in shadow mode**. If its performance metrics — precision, recall, F1 score — meet or exceed the current production model's, traffic is gradually shifted: 5%, then 25%, then 50%, then 100%, over a four-hour window. If at any point the challenger model's precision drops below threshold, traffic is automatically routed back to the champion.

The **retraining trigger** is automated: when our monitoring detects that model precision has dropped below **95%** or recall has dropped below **90%**, the Step Functions pipeline kicks off automatically.

### Layer 7: Monitoring and Observability

You cannot operate what you cannot observe. Our monitoring strategy covers three dimensions:

**System health** is tracked through **Amazon CloudWatch**. We monitor Lambda duration, Kinesis iterator age, SageMaker endpoint latency, and custom metrics such as score distribution and feature null rates.

**Centralized logging** flows to **CloudWatch Logs** from all Lambda functions and SageMaker endpoints, providing a unified view for debugging and incident response.

**Model performance** is tracked in **Amazon Timestream**, a purpose-built time-series database. We compute precision, recall, and F1 scores hourly on a labeled subset and run temporal queries to detect drift trends before they become critical.

We have defined five key alarms, each with a clear escalation path:

| Alarm | Trigger Condition | Automated Response |
|-------|-------------------|-------------------|
| Consumer lag | Kinesis iterator age exceeds 30 seconds | Page on-call engineer; scale Lambda concurrency |
| Inference latency | SageMaker p99 latency exceeds 50 ms | Scale out endpoint instances |
| Model drift | Precision drops below 95% | Trigger automated retraining pipeline |
| Data quality | Feature null rate exceeds 5% | Alert data engineering team |
| SLA risk | End-to-end p99 exceeds 150 ms | Page on-call engineer |

---

## End-to-End Data Flow

Let me walk through exactly what happens when a single transaction enters the system, with timing at each hop:

| Step | Component | Action | Latency (p50 / p99) |
|------|-----------|--------|---------------------|
| 1 | Client to API Gateway | HTTPS POST to /v1/transactions | 5 ms / 10 ms |
| 2 | API Gateway to Kinesis | PutRecord with 2 KB payload | 3 ms / 8 ms |
| 3 | Kinesis to Feature Engineer | Lambda trigger via event-source mapping | 8 ms / 15 ms |
| 4 | Feature Engineer to Redis | GET rolling aggregates | 1 ms / 2 ms |
| 5 | Feature Engineer to DynamoDB | GET customer profile | 3 ms / 5 ms |
| 6 | Feature Engineer | Compute derived features | 2 ms / 5 ms |
| 7 | Orchestrator to SageMaker | InvokeEndpoint with feature vector | 5 ms / 12 ms |
| 8 | Orchestrator to EventBridge | PutEvents with score and decision | 3 ms / 5 ms |
| | **Total (hot path)** | | **30 ms / 62 ms** |

At the median, we return a fraud decision in **30 milliseconds**. At the 99th percentile, **62 milliseconds**. Even at the 99.9th percentile, we project approximately **90 milliseconds** — well under our hard ceiling of 200 milliseconds. We have comfortable headroom across the board.

In parallel — and this is important — the warm path operates asynchronously and does not add any latency to the scoring decision:

1. Kinesis Firehose reads from the stream via enhanced fan-out.
2. Firehose buffers and writes compressed Parquet files to S3 every 60 seconds or 128 megabytes, whichever comes first.
3. Glue Crawlers update the catalog hourly.
4. Athena is available immediately for ad-hoc querying against the latest data.

---

## Key Performance Metrics

### Throughput Capacity

| Layer | Component | Target Throughput | How It Scales |
|-------|-----------|-------------------|---------------|
| Ingestion | API Gateway | 10,000 RPS | Regional limit increase |
| Ingestion | Kinesis | 10,000 TPS across 20 shards | Shard splitting |
| Feature Store | Lambda | 20 concurrent invocations | Provisioned concurrency |
| Feature Store | Redis | 200,000 ops/sec | Cluster mode with read replicas |
| Inference | SageMaker | 2,000 TPS per ml.c5.xlarge instance | Auto Scaling from 2 to 20 instances |
| Post-Processing | EventBridge | 10,000 events/sec | Fully managed, scales automatically |
| Data Lake | Firehose | 10,000 records/sec | Auto-scales |

Every layer has been sized to handle our sustained load of 5,000 TPS with capacity to absorb peaks of 25,000 TPS through scaling mechanisms that are either automatic or require a single API call.

### Latency Performance

| Percentile | End-to-End Latency | Budget | Status |
|------------|-------------------|--------|--------|
| p50 | ~30 ms | < 50 ms | Within budget |
| p95 | ~50 ms | < 80 ms | Within budget |
| p99 | ~62 ms | < 100 ms | Within budget |
| p99.9 | ~90 ms | < 200 ms | Within budget |

We are meeting all four latency targets with significant margin. At the p99, we are using 62% of our 100-millisecond budget, leaving 38 milliseconds of headroom for future feature additions or model complexity increases.

### Availability

The composite SLA across all serial dependencies in the hot path is calculated as follows:

| Component | Individual SLA |
|-----------|---------------|
| API Gateway | 99.95% |
| Kinesis Data Streams | 99.9% |
| Lambda | 99.95% |
| ElastiCache Redis | 99.9% (Multi-AZ) |
| SageMaker Endpoint | 99.9% |
| EventBridge | 99.99% |
| **Composite (product)** | **~99.6%** |

A composite SLA of 99.6% means approximately **35 hours of potential downtime per year** across the scoring path. That is not good enough on its own. Which is why we have built a **graceful degradation strategy**.

If SageMaker becomes unavailable, the orchestrator Lambda falls back to a **rule-based scoring engine** that is pre-loaded in Lambda memory. These rules cover the top 20 fraud patterns, which account for approximately **80% of all fraud detections**. The rule-based engine has a precision of roughly **85%** — lower than the model's 97%, but far better than no protection at all.

With this fallback in place, the effective availability of the scoring function rises to approximately **99.9%**, which meets our target.

---

## Cost Profile

Let me address the economics of this system. At a sustained throughput of 5,000 transactions per second, our estimated monthly costs are:

| Service | Configuration | Monthly Cost |
|---------|---------------|-------------|
| API Gateway | 13 billion requests/month | $1,500 |
| Kinesis Data Streams | 20 shards with enhanced fan-out | $800 |
| Lambda (Feature + Orchestrator) | 13 billion invocations, 128 MB memory, 50 ms average duration | $1,300 |
| ElastiCache Redis | 6x r6g.xlarge instances (Multi-AZ) | $1,600 |
| DynamoDB | On-demand, ~5,000 RCU average | $400 |
| SageMaker Endpoints | 4x ml.c5.xlarge instances (average) | $700 |
| Kinesis Firehose | 13 billion records to S3 | $200 |
| S3 (Data Lake) | ~50 TB/month (Parquet compressed) | $250 |
| CloudWatch and Timestream | Metrics, logs, alarms | $200 |
| Glue, Athena, Step Functions | ETL, queries, and workflows | $150 |
| **Total** | | **~$7,100/month** |

That is approximately **$85,000 per year** to score 400 million transactions daily in real time. To put that in perspective, that is roughly **$0.000018 per transaction** — less than two hundredths of a cent.

### Cost Optimization Levers

We have identified four levers that can further reduce costs as we mature the deployment:

1. **Compute Savings Plans**: A one-year commitment reduces Lambda and SageMaker costs by approximately 30%, saving **~$600 per month**.
2. **Kinesis On-Demand Mode**: Eliminates shard management entirely at roughly 20% higher cost — a reasonable trade-off if operational simplicity becomes a priority.
3. **SageMaker Spot Instances for Training**: We already plan to use spot instances for training jobs, achieving a **70% cost reduction** on training compute. We do not recommend spot for real-time inference endpoints due to interruption risk.
4. **S3 Intelligent-Tiering**: Automatically moves infrequently accessed data to cheaper storage tiers, saving approximately **40% on storage** after 90 days.

---

## Failure Modes and Resilience

No system is immune to failure. What matters is how we detect, contain, and recover from failures. I want to walk through the five failure scenarios we have designed for, because I believe our resilience posture is one of the strongest aspects of this architecture.

### Scenario 1: SageMaker Endpoint Unavailable

This could be caused by a deployment failure, a scaling event, or an availability zone outage. We detect it through the `Invocation5XXErrors` CloudWatch alarm and p99 latency spikes.

Our mitigation is the rule-based fallback engine I described earlier. It covers the top 20 fraud patterns and maintains an 85% precision rate. The SageMaker service auto-heals in most cases; if it does not, we can roll back to the previous model version through our deployment configuration.

The blast radius is reduced detection accuracy — not data loss, not system unavailability.

### Scenario 2: Redis Cluster Failure

A primary node failure or network partition in our ElastiCache cluster would be detected through the `ReplicationLag` alarm and connection errors in Lambda logs.

ElastiCache is configured for **Multi-AZ with automatic failover**, which completes in under **30 seconds**. During that window, the feature engineer Lambda falls back to DynamoDB for all reads. This increases per-lookup latency from under 1 millisecond to approximately 5 milliseconds — a noticeable but manageable impact.

After failover, Redis promotes a replica to primary and the cache warms organically from DynamoDB reads. No data is lost because DynamoDB is the source of truth.

### Scenario 3: Kinesis Throttling or Hot Shard

An uneven partition key distribution or traffic spike could cause writes to exceed a shard's capacity. We detect this through the `WriteProvisionedThroughputExceeded` metric and increasing iterator age.

Our partition key strategy uses a **random suffix** appended to the customer ID — for example, `customer_id + random(0-9)` — to distribute load evenly across shards. If throttling occurs, API Gateway returns a **429 status** with a `Retry-After` header, and the client implements exponential backoff.

Recovery involves **shard splitting**, which doubles capacity for the affected shard. No data is lost thanks to API Gateway retries and client-side backoff logic.

### Scenario 4: Model Drift

Fraud patterns evolve. Seasonal behavior shifts. Data distributions change. Over time, the model's performance will degrade if left unattended.

We detect drift through hourly precision, recall, and F1 computations stored in Timestream. When precision drops below **95%** or recall drops below **90%**, the retraining pipeline triggers automatically.

The new model is trained on recent data and deployed as a challenger serving 5% of traffic. Only after it passes validation does it get promoted to champion. This limits the blast radius: the worst case is a gradual increase in false positives or false negatives for up to 24 hours until the new model is deployed.

### Scenario 5: Lambda Cold Start Spike

After a quiet period or a deployment update, Lambda functions may experience cold starts that add 200 to 500 milliseconds of latency.

We mitigate this with **provisioned concurrency** — 20 instances for the feature engineer and 20 for the orchestrator. With provisioned concurrency in place, cold starts affect fewer than **0.1% of requests**. For Java runtimes, we additionally leverage **SnapStart** to reduce initialization time.

---

## Data Consistency Model

I want to address our data consistency approach transparently, because it involves deliberate trade-offs.

### Feature Store: Eventual Consistency

Our feature store operates under **eventual consistency**. Redis is updated synchronously by the feature engineer after each transaction, while DynamoDB is updated asynchronously for most features. This means there is a staleness window of up to approximately **1 second** where features in Redis may be ahead of DynamoDB.

This is acceptable for fraud detection because the model is trained on similarly delayed features. Consistency during that one-second window is not what catches fraud — it is the aggregate pattern across hundreds of features that drives detection accuracy.

### At-Least-Once Delivery and Deduplication

Kinesis provides at-least-once delivery, which means duplicate records can occur. We handle this at two levels:

- **In the hot path**: Each transaction carries an idempotency key. The feature engineer checks Redis for a recent processing record with a **5-minute TTL** before re-processing.
- **In the data lake**: Firehose may deliver duplicate records to S3. We apply deduplication during Glue ETL using a composite key of `transaction_id` and `timestamp` before preparing training data.

### Model Deployment Consistency

During model updates, we use **blue/green deployment** through SageMaker production variants:

1. The new model is deployed with 0% initial traffic weight.
2. Traffic shifts gradually: 5%, then 25%, then 50%, then 100%, over a four-hour window.
3. If precision drops below threshold at any point during ramp-up, traffic routes back to the previous model automatically.
4. Both models score the same feature vector format, so no feature store changes are required during model transitions.

### Compliance and Audit Trail

Every transaction and its corresponding fraud score are immutably stored in S3 with Object Lock in WORM compliance mode. DynamoDB Streams combined with Lambda archives all feature store mutations to S3 for regulatory replay capability. Athena provides the ability to query the complete decision history for any transaction within seconds. Our retention policy is **seven years**, configurable per regulatory jurisdiction.

---

## Summary and Recommendation

To summarize what we are proposing:

- A **seven-layer architecture** that cleanly separates real-time scoring, data archival, model training, and monitoring concerns.
- **Sub-100-millisecond fraud scoring** at the 99th percentile, with a hard ceiling of 200 milliseconds, on 400 million transactions per day.
- **99.9% effective availability** through a graceful degradation strategy that falls back to rule-based scoring when the ML endpoint is unavailable.
- **Automated model retraining** triggered by drift detection, with champion/challenger deployment that limits risk during model transitions.
- **Full regulatory compliance** with immutable transaction records, seven-year retention, and queryable audit trails.
- A total cost of approximately **$7,100 per month** — less than two hundredths of a cent per transaction.
- **Clear scaling paths** at every layer, from shard splitting to endpoint auto-scaling, that allow us to grow to 25,000 TPS and beyond without re-architecting.

This architecture is built entirely on managed AWS services, minimizing operational overhead. Every failure mode has a documented detection mechanism, mitigation strategy, and recovery path. And every design decision — from Kinesis over Kafka, to Lambda over ECS, to Redis over SageMaker Feature Store — has been made with explicit reasoning that we can revisit as our requirements evolve.

I am confident this design meets our security, performance, compliance, and cost objectives. I welcome your questions.

---

*Architecture diagram: see `real_time_fraud_detection_pipeline.png` in the pipeline directory.*
