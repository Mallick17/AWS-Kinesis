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

---

## Data Flow from Kinesis Data Streams → Final Destination
This section focuses **exclusively** on what happens **after** DMS has finished formatting the JSON record.  
```
[DMS Task: JSON record ready]
          ↓ (PutRecords API)
[Kinesis Data Streams: Durable buffer]
          ↓ (2 independent consumers)
[Consumer #1: Enhanced Fan-Out → Lambda → SNS]
[Consumer #2: Firehose → Lambda → S3 → Glue → Athena/Redshift/OpenSearch]
```

- **Latency**: 68 ms for alerts, 60–180 s for lake  
- **Throughput**: 0.39 MB/s ingest → 4 shards auto-scaled  
- **Cost Drivers**: Kinesis ingest $102/GB, retrieval $13.50/GB, Firehose $29/GB  
- **Use Case Example**: Real-time fraud alert + hourly analytics on every order change

<details>
    <summary>Click to view the breaking down every component, configuration, behavior, parameter, and edge case.</summary>

#### 1. Kinesis Data Streams: The Durable Buffer
- **What Happens**: Kinesis is a fully-managed, append-only log. DMS is the **producer**, Firehose & Lambda are **consumers**. Every record is immutable, sequence-numbered, and retained for exactly the period you set.
- **How Data Flows Here**:
  - DMS calls **PutRecords** (batch API, up to 500 records).  
  - Kinesis hashes the **PartitionKey** (`sales-orders-98765`) → lands in **shard-0003**.  
  - Record receives **SequenceNumber** `3962794871…` and **ApproximateArrivalTimestamp**.  
  - Kinesis writes the record **three times** across 3 AZs in < 500 ms.
  - **Behavior**:  
    - **Exactly-once ingest** (idempotent via sequence numbers).  
    - **At-least-once read** (consumers checkpoint their own progress).  
    - **Per-shard ordering** — records with the same PartitionKey never reorder.
- **Configuration for This Pipeline**:
  - **Stream Creation** (Console/CLI):
    ```bash
    aws kinesis create-stream \
      --stream-name rds-cdc-prod \
      --stream-mode-details StreamMode=ON_DEMAND \
      --retention-period-hours 168   # 7 days
    ```
  - **Encryption**: Server-Side Encryption with AWS KMS (customer-managed key).  
    Parameter: `--encryption-type KMS --kms-key-id alias/cdc-key`
  - **Monitoring**: Enhanced (shard-level) metrics enabled → CloudWatch every 1 min.
  - **IAM Role for DMS**:
    ```json
    {
      "Effect": "Allow",
      "Action": ["kinesis:PutRecords", "kinesis:PutRecord"],
      "Resource": "arn:aws:kinesis:*:*:stream/rds-cdc-prod"
    }
    ```
- **Why This Behavior?**  
  Kinesis is deliberately **decoupled** — DMS can burst to 10 MB/s, Kinesis auto-scales shards in 30 s, consumers read at their own pace.  
  If you change **PartitionKey** from `schema-table-id` to `random-uuid`, you eliminate hot shards but lose ordering — perfect for analytics, bad for replay.

- **Little Details & Varying Parameters**:
  - **On-Demand vs Provisioned**:
    - On-Demand: auto-scales to 200 MB/s, no shard management.  
    - Provisioned: you set 4 shards → $0.015/shard-hour = $108/month fixed.
  - **Retention**:
    - 24 h → free; 168 h (7 days) → $0.023/GB-month first 7 days, then $0.004.  
    - Edge case: set 365 days for GDPR replay → $0.40/GB-month.
  - **Hot Shard Mitigation**:
    - Bad key: `table=orders` → 1 shard 100 % hot.  
    - Good key: `table=orders||random(1000)` → even distribution.
  - **Throttling**:
    - CloudWatch `WriteProvisionedThroughputExceeded` → DMS retries with exponential back-off (1 s → 32 s).

#### 2. Consumer #1: Enhanced Fan-Out (Real-Time Path)
- **What Happens**: A **push** consumer that receives records in **70 ms** without polling.
- **Data Flow Behavior**:
  - You register the consumer once:
    ```bash
    aws kinesis register-stream-consumer \
      --stream-arn arn:aws:kinesis:...:rds-cdc-prod \
      --consumer-name fraud-alerts
    ```
  - Kinesis opens **HTTP/2** stream to your Lambda.  
  - Every record arrives **base64-encoded** with full metadata.  
  - Lambda runs **3-line** function → SNS → PagerDuty.
- **Configuration**:
  - **Lambda**:
    - Runtime: Node.js 20  
    - Memory: 128 MB  
    - Timeout: 5 s  
    - Trigger: Kinesis Enhanced Fan-Out (automatic).
  - **Concurrency**: Reserved 10 (pre-warmed).
- **Why 70 ms?**  
  No polling → Kinesis pushes directly over persistent connection.  
  Change to **standard fan-out** → latency jumps to 200–1000 ms.

#### 3. Consumer #2: Kinesis Data Firehose (Lake Path)
- **What Happens**: Firehose is a **serverless buffer + delivery** service. It is the only consumer that writes to S3.
- **Data Flow Behavior**:
  1. Firehose polls the stream every 1 s (standard fan-out).  
  2. Buffers in memory → 64 MB **or** 60 s (whichever first).  
  3. Invokes **Lambda transform** → JSON → Parquet + Snappy.  
  4. Writes **partitioned** objects to S3.  
  5. Emits **manifest** file.  
  6. EventBridge → Glue Crawler → Athena table updated.
- **Configuration** (one CLI command):
  ```bash
  aws firehose create-delivery-stream \
    --delivery-stream-name rds-to-lake \
    --kinesis-stream-source-configuration \
        StreamARN=arn:aws:kinesis:...:rds-cdc-prod,RoleARN=arn:... \
    --extended-s3-destination-configuration \
        BucketARN=arn:aws:s3:::cdc-lake-prod \
        Prefix=raw/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/ \
        BufferingHints='{"SizeInMBs":64,"IntervalInSeconds":60}' \
        CompressionFormat=Snappy \
        ProcessingConfiguration='{"Enabled":true,"Processors":[{"Type":"Lambda","Parameters":[{"ParameterName":"LambdaArn","ParameterValue":"arn:...:cdc-parquet"}]}]}' \
        ErrorOutputPrefix=errors/ \
        DataFormatConversionConfiguration='{"Enabled":true,"InputFormat":"JSON","OutputFormat":"PARQUET"}'
  ```
- **Lambda Transform** (8 lines):
  ```js
  exports.handler = async (event) => {
    return event.records.map(r => ({
      recordId: r.recordId,
      result: 'Ok',
      data: Buffer.from(JSON.stringify({
        ...JSON.parse(Buffer.from(r.data, 'base64')),
        processed_at: new Date().toISOString()
      }))).toString('base64')
    }));
  };
  ```
- **Why This Behavior?**  
  Firehose guarantees **exactly-once** to S3 (idempotent PUTs).  
  If you set `IntervalInSeconds=30` → latency halves, cost rises 8 %.

- **Little Details & Varying Parameters**:
  - **Back-pressure**: S3 throttles → Firehose retries 24 h, then DLQ.  
  - **Error Handling**: Bad record → `ProcessingFailed` → S3 `errors/` prefix + SNS.  
  - **Dynamic Partitioning**: `!{partitionKeyFromLambda:path}` → per-table folders.  
  - **2025 Feature**: Native Parquet conversion (no Lambda) for <10 ms transform.

#### 4. Final Destinations
- **S3 + Glue + Athena**:
  - Objects: `part-00000.snappy.parquet`  
  - Glue Crawler runs every 5 min → `CREATE EXTERNAL TABLE cdc.orders`.  
  - Query: `SELECT * FROM cdc.orders WHERE id=98765;` → 2 s.
- **Redshift**:
  - Spectrum COPY from manifest → zero-duplicate upserts via staging table.
- **OpenSearch**:
  - Firehose direct destination → `_bulk` API → upsert script on `id`.

#### Mini-Checklist (copy-paste into CloudWatch)
```
[x] PutRecords → 1.38 MB
[x] EFO → Lambda 68 ms
[x] Firehose buffer → 60 s
[x] Lambda → Parquet+Snappy
[x] S3 PUT → part-00000.snappy.parquet
[x] Glue Crawler → Athena table updated
[x] Redshift COPY → 0 duplicates
[x] IteratorAgeMilliseconds = 87 ms
```

#### One-Page Runbook
```markdown
# Kinesis → Lake
DMS → PutRecords → shard-0003
EFO → fraud-alert Lambda (68 ms)
Firehose → 60 s buffer → Lambda → Parquet
S3 → part-*.snappy.parquet + manifest
Glue Crawler → Athena/Redshift/OpenSearch
Queryable in < 3 s
```

From here, the data is sent to S3, Redshift, OpenSearch, or any other services.

</details>

---

### Amazon KDS Pricing
