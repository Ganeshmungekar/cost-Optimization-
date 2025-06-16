Cost Optimization Challenge: Managing Billing Records in Azure Serverless Architecture

🧩 Problem Summary

A serverless billing service stores over 2 million records (each up to 300 KB) in Azure Cosmos DB. These records grow in volume every year, making storage increasingly costly. However, older records (older than 3 months) are rarely accessed.

🎯 Optimization Goals

Reduce storage cost for rarely accessed records

Maintain availability with a few seconds latency

No downtime, no data loss

Keep existing API contracts unchanged

Simple to deploy and maintain



---

✅ Enhanced Solution: Tiered Storage with Cold Retrieval and Resiliency

Key Components:

Azure Cosmos DB → Stores active records (last 3 months)

Azure Blob Storage (Cool or Archive Tier) → Stores archived records

Azure Durable Functions → Handles data movement, retries, and resiliency

Azure Redis Cache → Caches frequently accessed archived records

Fallback Read Layer → Tries Cosmos first, then cold archive

Blob Index Tags & Partitioned Storage → Organize archived data efficiently



---

🏗️ Architecture Diagram (Text View)

Client → API Gateway
        ↓
    Azure Function (Read/Write)
        ↓
     Cosmos DB (Hot Data < 3 months)
        ↓      ↘             ↘
     Not Found Azure Redis   Azure Blob (Cold Data > 3 months)
                       ↑         ↑
                 Fallback    Durable Function (Archival Jobs)


---

🔥 Real-Time Challenges & Smart Solutions

1. Hot-Path Latency During Fallback

🔍 Issue: Decompressing large blobs during reads causes latency

✅ Fix: Use monthly-partitioned blob files like billing/2024/03/data.json.gz and cache results in Azure Redis


2. Partial Archival Failures

🔍 Issue: Mid-process crashes leave records inconsistent

✅ Fix: Use Durable Functions to checkpoint progress and mark records with a "status": "archived" flag before final deletion


3. Schema Evolution in Archived Records

🔍 Issue: API expects new fields not present in old data

✅ Fix: Read fallback wrapper uses defaults for missing fields (e.g., record.setdefault("discount", 0.0))


4. Blob Storage Failures

🔍 Issue: Blob download fails temporarily

✅ Fix: Add retry + exponential backoff; emit telemetry to Application Insights; enqueue failed lookups for retry


5. Audit & Compliance

🔍 Issue: Need traceability of record movement & access

✅ Fix: Log all archival and fallback reads with timestamps, user IDs, and operation results to Azure Monitor or Log Analytics



---

🔧 Core Implementation

Data Archival Logic

# Archive records older than 90 days to Blob
# Durable Function with checkpointing
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobClient
import json, gzip

COSMOS_CONN = "<cosmos-connection>"
BLOB_CONN = "<blob-connection>"

client = CosmosClient.from_connection_string(COSMOS_CONN)
container = client.get_database_client("BillingDB").get_container_client("Records")

month_key = get_archive_key()  # e.g. billing/2024/03/archive.json.gz
blob = BlobClient.from_connection_string(BLOB_CONN, "billing-archive", month_key)

records = list(container.query_items(
    query="SELECT * FROM Records r WHERE r.timestamp < @cutoff",
    parameters=[{"name": "@cutoff", "value": get_cutoff_date()}],
    enable_cross_partition_query=True
))

data = json.dumps(records).encode('utf-8')
blob.upload_blob(gzip.compress(data), overwrite=True)

for r in records:
    container.delete_item(r['id'], r['partitionKey'])


---

Read Wrapper with Fallback and Caching

import redis, gzip, json
from azure.storage.blob import BlobClient

redis_cache = redis.StrictRedis(host='cache.redis', port=6379)

# Try reading from Cosmos
try:
    return cosmos_container.read_item(item=record_id, partition_key=record_id)
except:
    cached = redis_cache.get(record_id)
    if cached:
        return json.loads(cached)

    # Decompress and search blob
    blob = BlobClient.from_connection_string(BLOB_CONN, container_name="billing-archive", blob_name=lookup_month_blob(record_id))
    records = json.loads(gzip.decompress(blob.download_blob().readall()))
    found = next((r for r in records if r['id'] == record_id), None)
    if found:
        redis_cache.setex(record_id, 3600, json.dumps(found))  # Cache for 1 hour
    return found


---

🎁 Bonus Features

Feature	Benefit

Blob Lifecycle Policy	Auto-transition blobs to Archive tier after 90 days
Event Grid + Metadata TTL	Trigger archival dynamically as new records arrive
Application Insights	Monitor performance and blob fallback success rate
Soft Deletes	Prevent accidental data loss during migration
Monthly Cost Dashboard	Logic App or Power BI pulls Cosmos + Blob usage



---

🔗 GitHub Repository

🛠 Please create this repo: https://github.com/ganeshmungekar/azure-billing-optimizer

Structure suggestion:

📁 billing-optimizer
├── archive_function/
├── read_wrapper_function/
├── templates/ (bicep/terraform)
├── README.md
└── tests/


---

✅ Summary

This advanced yet easy-to-implement architecture uses:

Hot + cold storage tiering

Caching + fallback logic

Resilience for failures & audit compliance

All while preserving API contracts and incurring no downtime


Perfect for interviews and real-world production use.
