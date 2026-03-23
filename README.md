# Real-Time Telematics Pipeline — AWS Kinesis to Delta Lake

A real-time data pipeline that ingests telematics data from AWS Kinesis into a Databricks Delta Lake Bronze table using Delta Live Tables (DLT).

---

## Architecture

```
Producer (Parquet / IoT Device)
        ↓
AWS Kinesis Stream
        ↓
Databricks DLT Pipeline
        ↓
Bronze Delta Table (telematics)
```

---

## Tech Stack

- **AWS Kinesis** — Real-time data streaming
- **Databricks DLT** — Pipeline orchestration
- **Apache Spark (PySpark)** — Data transformation
- **Delta Lake** — Data storage
- **AWS IAM + Unity Catalog** — Secure credential management

---

## Project Structure

```
kinesis_ingestion/
├── transformations/
│   └── kinesis_parsed.py       # DLT pipeline — reads Kinesis, writes to Bronze table
├── kinesis_producer.py         # Pushes parquet data into Kinesis for testing
└── README.md
```

---

## Pipeline Details

### Bronze Table — `telematics`

Reads raw JSON data from Kinesis stream, decodes it and stores into a Delta table with the following schema:

| Column | Type | Description |
|---|---|---|
| chassis_no | String | Vehicle identifier |
| latitude | Double | GPS latitude |
| longitude | Double | GPS longitude |
| event_timestamp | String | Event time |
| speed | Double | Vehicle speed |
| stream_metadata | Struct | Kinesis metadata (shardId, sequenceNumber, etc.) |

---

## Setup

### 1. AWS IAM Role Configuration

Create an IAM Role with the following **permissions policy**:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "kinesis:DescribeStream",
                "kinesis:DescribeStreamSummary",
                "kinesis:ListShards",
                "kinesis:GetShardIterator",
                "kinesis:GetRecords",
                "kinesis:PutRecord",
                "kinesis:PutRecords",
                "kinesis:ListStreams"
            ],
            "Resource": "arn:aws:kinesis:<region>:<account-id>:stream/<stream-name>"
        }
    ]
}
```

And the following **trust policy** (required by Unity Catalog):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::414351767826:role/unity-catalog-prod-UCMasterRole-14S5ZJVKOTYTL"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "sts:ExternalId": "<your-unity-catalog-credential-external-id>"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<account-id>:role/<role-name>"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

> **Note:** The second statement (self-assume) is required by Unity Catalog.

### 2. Unity Catalog Credential

Create a service credential in Databricks:
```
Catalog Explorer → Credentials → Create Credential
```
Note down the **External ID** and use it in the trust policy above.

### 3. Kinesis Config in Pipeline

```python
kinesis_config = {
    "streamName": "<your-stream-name>",
    "region": "<your-region>",
    "serviceCredential": "<your-unity-catalog-credential>",
    "initialPosition": "trim_horizon"
}
```

---

## Key Learnings

- Unity Catalog credentials require the IAM role to **self-assume**
- Kinesis `data` column must be cast to string before JSON parsing
- Use `StructType` instead of `MapType` for proper column types
- `trim_horizon` reads from the beginning of the stream (equivalent to `earliest` in Kafka)
- Always use **Full Refresh** in DLT when changing table schema

---

## Output

40 telematics records successfully ingested into Bronze Delta table.

![Pipeline Screenshot](pipeline_screenshot.png)
