# AWS-Kinesis
Amazon Kinesis is a platform of services provided by Amazon Web Services (AWS) for collecting, processing, and analyzing real-time streaming data at a large scale. It allows for the ingestion of various types of data, such as application logs, website clickstreams, and IoT telemetry data, enabling real-time analytics and insights.
- The Kinesis platform comprises several distinct services tailored for specific use cases:

<details>
    <summary>Click to view the several distinct services</summary>

> 1. Amazon Kinesis Data Streams
  - This service enables the building of custom applications that process or analyze streaming data. It handles the ingestion and storage of streaming data in shards, allowing for replay and consumption during a defined retention period.

> 2. Amazon Kinesis Video Streams
  - This service focuses on building custom applications that process or analyze streaming video. It allows for ingesting live video streams from various sources and building applications to access and process the data frame-by-frame in real-time or for batch processing of historical data.

> 3. Amazon Data Firehose
  - This service is designed for delivering real-time streaming data to various AWS destinations, including Amazon S3, Amazon Redshift, Amazon OpenSearch Service, and Splunk. It simplifies the process of loading streaming data into these destinations without requiring custom application development.

> 4. Amazon Managed Service for Apache Flink (formerly Amazon Kinesis Data Analytics)
  - This service allows for processing and analyzing streaming data using standard SQL or with Java (managed Apache Flink). It enables the creation of applications that continuously read and process streaming data, perform time-series analytics, feed real-time dashboards, and generate real-time metrics. 

</details>

## Flow of Data from RDS --> CDC --> Kinesis Data Streams --> Data Firehose

### Comprehensive Breakdown: Data Flow from RDS via DMS (CDC) to Kinesis Data Streams, Then to Kinesis Data Firehose
Think of this pipeline as a **real-time Change Data Capture (CDC) conveyor belt**: It captures database changes (inserts, updates, deletes) from RDS, streams them durably, and loads them into a destination like S3 for analytics. It's fault-tolerant, scalable, and low-latency (~seconds end-to-end), but requires careful tuning to avoid bottlenecks.

<details>
    <summary>Click to view the detailed explaination</summary>

#### High-Level Pipeline Diagram (Text-Based)
```
[Source: RDS (e.g., MySQL/PostgreSQL)] 
    ↓ (CDC via binary logs/supplemental logs)
[AWS DMS Task: Full Load + Ongoing Replication]
    ↓ (JSON-formatted records with metadata)
[Kinesis Data Streams: Durable Buffer/Stream]
    ↓ (Consumer reads: e.g., via Lambda or direct integration)
[Kinesis Data Firehose: ETL + Delivery Stream]
    ↓ (Batching, transformation, compression)
[Destination: e.g., S3, Redshift, Elasticsearch]
```
- **Latency**: DMS to Streams: <1s; Streams to Firehose: <1s; Firehose to S3: 60s-5min (batch window).
- **Throughput**: Scales with shards (Streams) and buffer size (Firehose).
- **Cost Drivers**: DMS replication instance hours + Kinesis ingest/read GB + Firehose delivery GB.
- **Use Case Example**: E-commerce RDS tracking order changes → Real-time fraud alerts (via Streams consumers) + daily analytics in S3 (via Firehose).

#### 1. Source: Amazon RDS with CDC Setup
- **What Happens**: RDS (e.g., MySQL 8.0, PostgreSQL 15, Aurora) acts as the source database. CDC captures **changes** (DML: INSERT/UPDATE/DELETE; sometimes DDL) in near real-time using database-native mechanisms.
- **How CDC Works Here**:
  - **MySQL/Aurora MySQL**: Enable binary logging (`binlog_format=ROW`, `binlog_row_image=FULL`) in RDS parameter group. DMS reads the binlog trail for changes.
  - **PostgreSQL/Aurora PostgreSQL**: Use logical replication (enable `rds.logical_replication=1` in parameter group). DMS subscribes to WAL (Write-Ahead Log) via pglogical or native slots.
  - **Behavior**: Changes are captured as **before/after images** (e.g., old/new row values) with metadata (timestamp, operation type: I/U/D, LOB handling). Full load first dumps existing tables, then switches to CDC.
  - **Little Details**:
    - **LOBS (Large Objects)**: DMS handles up to 1 GB; configure `LobMaxSize=0` for full LOBs (risks memory spikes).
    - **Transactions**: DMS preserves atomicity—partial txns are buffered until commit.
    - **Error Handling**: If RDS lags (e.g., high CPU), DMS retries (configurable: `MaxFullLoadSubTasks`). Failures resume from last checkpoint (stored in DMS metadata DB).
- **Configuration for This Pipeline**:
  - RDS Instance: Multi-AZ for HA; enable backups/performance insights.
  - Parameter Group: As above + `max_connections` tuned for DMS load (e.g., +20% for replication slot).
  - **Why This Behavior?**: CDC is log-based (not polling) for efficiency—low overhead (<5% CPU on source). If params change (e.g., `binlog_format=STATEMENT`), DMS falls back to less granular capture, losing row-level details.
- **Varying Parameters Example**:
  - High-volume RDS (1M txns/hr): Increase `binlog_cache_size` to avoid OOM; behavior shifts to more frequent log rotations, increasing DMS read latency by 10-20%.
  - Serverless Aurora (2025 update): Auto-scales compute; DMS uses "serverless" endpoints, but CDC slots can pause during scale-down, causing 1-5min gaps.

#### 2. AWS DMS: CDC Extraction and Delivery to Kinesis Data Streams
- **What Happens**: DMS is the **orchestrator**—a managed ETL service that reads CDC from RDS and writes to Kinesis as JSON records. Supports full load (initial snapshot) + ongoing CDC.
- **Data Flow Behavior**:
  - DMS creates a **replication instance** (EC2-based, e.g., t3.medium) that tails the RDS log.
  - Records are transformed to **JSON** (default) with schema: `{ "metadata": { "timestamp", "operation", "partition-key" }, "data": { "before": {}, "after": {} } }`.
  - **Full Load Phase**: Table-by-table dump (parallel via sub-tasks); then "CDC only" mode.
  - **CDC Phase**: Ongoing changes streamed every ~1s; DMS batches 1K-10K records before PutRecord to Kinesis.
  - **Latency**: End-to-end <2s; tunable via task settings.
- **Configuration for Kinesis Target** :
  - **Endpoint Settings** (in DMS console/CLI):
    - `ServiceAccessRoleArn`: IAM role with `AmazonKinesisFullAccess` + trust for DMS.
    - `StreamArn`: Your Kinesis stream ARN (e.g., arn:aws:kinesis:us-east-1:123:stream/my-cdc-stream).
    - `DataFormat`: JSON (default); or Apache Avro for schema evolution.
    - `IncludeTableAlterOperations`: True for DDL changes (e.g., ALTER TABLE → metadata event).
    - `PartitionKeyType`: GLOBAL (single stream) or PRIMARY_KEY (shards by PK for even distribution).
    - `IncludePartitionValue`: True to embed PK in JSON for downstream routing.
    - `TimestampColumnName`: e.g., "change_timestamp" for ordering.
  - **Task Settings**:
    - `TargetMetadata`: `{ "TargetSchema": "", "SupportLobs": true, "LobMaxSize": 64MB, "LimitedSizeLobMode": false }`.
    - `FullLoadSettings`: `{ "MaxFullLoadSubTasks": 8, "TransactionConsistencyTimeout": 600 }` (waits for txns to commit).
    - `CdcSettings`: `{ "EnableBitmapAttachments": false }` (avoids extra binary data).
    - Table Mapping: JSON rules like `{"rules": [{"rule-type":"selection","rule-id":"1","rule-name":"1","object-locator":{"schema-name":"%","table-name":"%"},"rule-action":"include"}]}`.
  - **Validation**: Enable `TaskValidationSettings` for post-load checks (e.g., row counts).
- **Why This Behavior?**:
  - DMS is **decoupled**—source failures don't crash the task; it resumes from LSN (Log Sequence Number) or SCN (System Change Number).
  - Batching reduces Kinesis PutRecord calls (costs $0.014/1K records).
  - **Hot Partitions**: If `PartitionKeyType=PRIMARY_KEY` and skewed PKs (e.g., timestamp-based), uneven shard load → throttling.
- **Little Details & Varying Parameters** [web:4, web:7]:
  - **Throughput Tuning**: `BatchApplyEnabled=true` + `MinFileSize=100MB` increases CDC speed by 2x but spikes memory (monitor via CloudWatch: `CDCLatencySource`, `CDCLatencyTarget`).
    - Example: Default (1 task): 10K records/s. Set `ParallelLoadThreads=16` → 50K/s, but risks RDS overload.
  - **Error Modes**: `ErrorMaxCount=10` → task stops on 11th error; set to 0 for infinite retries (but logs bloat).
  - **2025 Update**: Serverless DMS (preview) auto-scales tasks; for Aurora Serverless v2, CDC behaves with zero-downtime scaling, but `CdcMaxBatchInterval=5s` may cause bursts .
  - If params change: High LOBs (`LobMaxSize=0`) → DMS chunks to 50MB, increasing record count 20x, amplifying Kinesis ingest costs.

#### 3. Kinesis Data Streams: The Core Durable Buffer (Deep Dive: Every Little Bit)
This is the **heart** of the pipeline—a partitioned, append-only log for real-time streaming. In CDC, it buffers DMS records durably (99.9% SLA), allowing multiple consumers (e.g., Firehose) to read independently. Data is stored as **immutable records** with sequence numbers; no updates/deletes.

- **Core Concepts & Behavior** :
  - **Stream**: A named collection (e.g., "rds-cdc-stream"). Data partitioned into **shards** (units of parallelism/capacity).
  - **Shard**: Fixed throughput pipe. Ingest: 1 MB/s writes, 1K records/s max. Read: 2 MB/s shared (standard fan-out) or 2 MB/s dedicated (enhanced).
    - **Sequence Numbers**: Per-shard, strictly increasing (e.g., 1234567890). Ensures ordering within shard; cross-shard ordering via timestamps.
    - **Retention**: 24h default (up to 365 days); data expires atomically per record.
  - **Record**: Up to 1 MB JSON from DMS. Includes **approximate arrival timestamp** (millis precision).
  - **Data Flow in This Pipeline**: DMS does `PutRecord`/`PutRecords` (batched). Records land in shards based on **partition key** (e.g., table name or PK from DMS config). Firehose (or Lambda) consumes via `GetRecords`.
  - **Behavior Quirks**:
    - **At-Least-Once Delivery**: Duplicates possible on retries (use idempotency keys: `AggregationId`).
    - **Ordering**: Per-shard only—use partition keys wisely for logical grouping (e.g., by database/table).
    - **Throttling**: Hits if >1 MB/s ingest/shard → `ProvisionedThroughputExceededException`. CloudWatch: `WriteProvisionedThroughputExceeded`.
    - **Hot Shards**: Uneven distribution (e.g., all changes to one table) → some shards idle, others throttle. Mitigate: Randomize keys or use on-demand mode .
    - **Resharding**: Split/merge shards dynamically (API: `SplitShard`). Behavior: New child shards start fresh sequence numbers; traffic splits 50/50.

- **Capacity Modes & Scaling Behavior**:
  - **Provisioned**: Manual shard count. Calc: Shards = ceil( (records/s * size KB) / 1024 / 1 MB/s ).
    - Example: DMS CDC at 100 records/s, 4 KB each → 0.39 MB/s → 1 shard.
    - Behavior: Fixed cost ($0.015/shard-hour); over-provision → waste, under → throttle (retries add 100-500ms latency).
    - Vary: Spike to 1K records/s → Reshard to 4; auto-alarm via CloudWatch if `IncomingBytes` >80% capacity.
  - **On-Demand** (default for new streams, 2025): Auto-scales shards (up to 1K active). Behavior: Instant ramp (seconds); pays per GB ($0.1023 ingested).
    - Example: CDC burst (e.g., batch job) → Auto-provisions 2x shards temporarily; settles to 1.
    - Why? Elastic for variable CDC loads (e.g., EOD reports). Downside: 25% pricier for steady traffic.
    - Vary: High variance (e.g., hourly peaks) → On-demand saves manual ops; but monitor `ShardCount` to avoid "soft limits" (500 shards default, request increase).

- **Consumers & Fan-Out: Reading Behavior** [web:22, web:23, web:27]:
  - **Consumers**: Apps reading data (e.g., Firehose as a "consumer"). Use Kinesis Client Library (KCL) for multi-threaded shard assignment.
    - **Standard Fan-Out**: Shared 2 MB/s per shard across all consumers (up to ~5 concurrent). Pull-based: Poll `GetRecords` every 5s.
      - Behavior: Competing consumers → latency spikes (e.g., 200ms → 1s if 3 consumers). Iterator age (CloudWatch: `GetRecords.IteratorAgeMilliseconds`) >15min → lag.
      - Example: DMS → 1 shard; 2 consumers (alert app + Firehose) → Each gets ~1 MB/s; fine for <2.
    - **Enhanced Fan-Out**: Dedicated 2 MB/s per consumer (up to 20/stream). Push-based: Kinesis calls back via HTTP/2.
      - Behavior: ~70ms P99 latency (vs. 200ms standard); no sharing—ideal for CDC real-time (e.g., fraud detection).
      - Config: Register consumer ARN via `RegisterStreamConsumer`; subscribe shards.
      - Why? Isolates traffic; but $0.013/GB extra retrieval. Vary: >2 consumers → Enable; for 1 (just Firehose), standard saves 50% cost.
    - **Little Details**: KCL v2 uses DynamoDB for checkpointing (last read position). Consumer group like Kafka—multiple instances share shards round-robin.
      - Vary: Skewed reads (one consumer hogs) → Use EFO to dedicate.

- **Retention & Time-Based Behavior**:
  - Default 24h; extend to 7/30/365 days ($0.023/GB-month first 7 days, $0.004 after).
  - Behavior: Expired data unreadable; use for CDC replay (e.g., fix consumer bugs). In pipeline: 1 day suffices for Firehose buffering.
  - Vary: 365 days for compliance → Costs 10x; but enables long-tail analytics.

- **Encryption & Security**:
  - Default: No encryption; enable SSE-KMS ($0.01/GB). Behavior: Transparent; keys rotate auto.
  - IAM: Producers (DMS) need `kinesis:Put*`; consumers `kinesis:Get*`, `kinesis:Subscribe*` (EFO).

- **Monitoring & Little Behaviors** :
  - CloudWatch Metrics: Per-stream (`IncomingRecords`), per-shard (`GetRecords.Bytes`).
  - Alarms: `IteratorAge > 1h` → Lag; `SuccessfulGetRecordCount=0` → Idle consumer.
  - 2025: Enhanced metrics for on-demand (shard utilization %).

- **CDC-Specific Behaviors**:
  - DMS records include op metadata—Streams preserves it for downstream (e.g., Firehose transforms U→upsert).
  - If DMS lags: Streams buffers (up to retention); but high volume → backpressure to DMS (configurable `DmsTransferEndpoint`).

- **Why All This?**: Streams is **durable FIFO queue**—decouples producers/consumers, enables fan-out (one write, many reads). Parameters balance cost/latency/scale; poor tuning (e.g., bad keys) causes 90% of issues.

#### 4. From Kinesis Data Streams to Kinesis Data Firehose: Integration & Delivery
- **What Happens**: Firehose acts as a **serverless ETL buffer** consuming from Streams, transforming (optional), and delivering to sinks. Direct integration: Firehose sources include "Kinesis Data Stream" .
- **Data Flow Behavior**:
  - A **consumer** (e.g., Lambda or Firehose's built-in) reads from Streams via GetRecords.
  - Firehose batches (60s-5min window), compresses (GZIP), and PUTs to destination.
  - **Transformation**: Optional Lambda (e.g., filter CDC deletes, enrich with timestamps).
  - Latency: Streams read <1s + batch 1-5min → S3.
- **Configuration** :
  - Create Firehose Delivery Stream: Source = Kinesis Stream ARN; Destination = S3 bucket.
  - `BufferSize=128MB`, `BufferInterval=300s` (tune for volume: smaller = lower latency, higher cost).
  - IAM Role: `FirehoseFullAccess` + S3 write.
  - Processing: Lambda ARN for CDC-specific (e.g., convert JSON to Parquet).
- **Why This?**: Firehose adds **durability + ETL**—Streams is raw buffer, Firehose handles delivery failures (retries 24h).
- **Little Details & Varying**:
  - **Backpressure**: If S3 throttles, Firehose buffers in-memory (up to 128MB); overflow → error delivery stream.
  - Vary: High CDC volume → `BufferInterval=60s` reduces staleness but increases PUTs ($0.029/GB out).
  - Integration Quirk: Firehose uses standard fan-out; for low-latency, insert Lambda with EFO.

#### Final Notes: End-to-End Tuning & Tradeoffs
- **Full Pipeline Cost Example**: For 100 records/s CDC: DMS ~$50/mo (t3.small), Streams $140 (as prior), Firehose $20 (1GB/day to S3) → ~$210/mo.
- **Testing**: Use DMS validation + Kinesis sample apps. Monitor end-to-end with X-Ray.
- **Edge Cases**: Cross-Region? Use replication. Failover? DMS multi-task + Streams enhanced HA.

</details>

---

### Amazon KDS Pricing
