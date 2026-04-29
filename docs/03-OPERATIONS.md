# Disease Search Operations: Setup, Testing & Troubleshooting

## Table of Contents

1. [Local Development Setup](#local-development-setup)
2. [Search Query Examples](#search-query-examples)
3. [Testing & Validation](#testing--validation)
4. [Troubleshooting](#troubleshooting)
5. [Production Deployment](#production-deployment)
6. [Monitoring & Performance](#monitoring--performance)

---

## Local Development Setup

### Prerequisites

- Java 17
- Maven
- Docker (for Elasticsearch)
- Postman or curl

### Step 1: Start Elasticsearch Locally

```bash
# Start single-node Elasticsearch 8.11.0
docker run --name elasticsearch \
  --rm \
  -p 9200:9200 \
  -e discovery.type=single-node \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  -e "xpack.security.enabled=false" \
  docker.elastic.co/elasticsearch/elasticsearch:8.11.0

# Wait for message: "Elasticsearch started"
```

### Step 2: Verify Elasticsearch is Running

```bash
# Check cluster health
curl -X GET "http://localhost:9200/_cluster/health"

# Expected response:
{
  "cluster_name": "docker-cluster",
  "status": "green",
  "timed_out": false,
  "number_of_nodes": 1,
  "number_of_data_nodes": 1,
  "active_primary_shards": 0,
  "active_shards": 0,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 0
}
```

### Step 3: Configure Spring Boot

**application-dev.yml:**

```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
    socket-timeout: 10s
    connect-timeout: 5s

app:
  search:
    disease:
      elasticsearch:
        enabled: true

logging:
  level:
    org.springframework.data.elasticsearch: DEBUG
    com.artemhealthtech: DEBUG
```

### Step 4: Build and Run Application

```bash
cd bmc-user-api

# Build
mvn clean package

# Run
mvn spring-boot:run -Dspring-boot.run.arguments="--spring.profiles.active=dev"

# Watch for:
# ✓ "Checking Elasticsearch index: diseases_v1"
# ✓ "Index created successfully: diseases_v1"
# ✓ "ApplicationReadyEvent fired"
```

### Step 5: Bulk Reindex Diseases

```bash
# Reindex all diseases into Elasticsearch
# Call admin endpoint (create one in your controller)
POST http://localhost:8080/admin/disease/reindex

# Response:
{
  "status": "success",
  "totalIndexed": 5000,
  "duration": "2.5s"
}

# Or do it manually:
curl -X POST "http://localhost:9200/diseases_v1/_refresh"
```

---

## Search Query Examples

### Using Kibana Dev Console

#### 1. Open Kibana

```
http://localhost:5601
```

Navigate to: **Dev Tools** → **Console**

#### 2. View All Documents

```
GET /diseases_v1/_search
{
  "size": 10,
  "query": {
    "match_all": {}
  }
}
```

#### 3. View Single Document

```
GET /diseases_v1/_doc/1
```

**Response:**

```json
{
  "_index": "diseases_v1",
  "_id": "1",
  "_version": 1,
  "_seq_no": 0,
  "_primary_term": 1,
  "found": true,
  "_source": {
    "id": 1,
    "diseaseName": "Fever",
    "diseaseCode": "FVR",
    "diseaseCodeNormalized": "fvr",
    "categoryId": 10,
    "categoryName": "Infectious Diseases",
    "subCategoryId": 20,
    "subCategoryName": "Viral Infections",
    "healthStandardId": 5,
    "healthStandardName": "WHO Guidelines",
    "isActive": true
  }
}
```

### Using Spring Boot API

#### 1. Search for "fever"

**Request:**

```bash
POST http://localhost:8080/api/v1/user/disease/page
Content-Type: application/json

{
  "searchKey": "fever",
  "page": 0,
  "size": 10
}
```

**Response:**

```json
{
  "content": [
    {
      "id": 1,
      "diseaseName": "Fever",
      "diseaseCode": "FVR",
      "diseaseCodeNormalized": "fvr",
      "categoryId": 10,
      "categoryName": "Infectious Diseases",
      "subCategoryId": 20,
      "subCategoryName": "Viral Infections",
      "healthStandardId": 5,
      "healthStandardName": "WHO Guidelines",
      "isActive": true
    },
    {
      "id": 42,
      "diseaseName": "Scarlet Fever",
      "diseaseCode": "SF",
      "categoryId": 10,
      "categoryName": "Infectious Diseases",
      ...
    }
  ],
  "totalElements": 156,
  "totalPages": 16,
  "currentPage": 0,
  "pageSize": 10,
  "numberOfElements": 10,
  "last": false
}
```

#### 2. Search with Pagination

```bash
# Page 1 (first 10)
POST http://localhost:8080/api/v1/user/disease/page
{
  "searchKey": "fever",
  "page": 0,
  "size": 10
}

# Page 2 (next 10)
POST http://localhost:8080/api/v1/user/disease/page
{
  "searchKey": "fever",
  "page": 1,
  "size": 10
}

# Page 3 (next 10)
POST http://localhost:8080/api/v1/user/disease/page
{
  "searchKey": "fever",
  "page": 2,
  "size": 10
}

# RESULT: Each page has 10 UNIQUE diseases (no duplicates like SQL)
```

#### 3. Typo-Tolerant Search

```bash
# User types "feavor" (misspelling of "fever")
POST http://localhost:8080/api/v1/user/disease/page
{
  "searchKey": "feavor",
  "page": 0,
  "size": 10
}

# RESULT: Still finds "Fever" via fuzzy matching
```

#### 4. Code Search

```bash
# Search for disease by code
POST http://localhost:8080/api/v1/user/disease/page
{
  "searchKey": "S82",
  "page": 0,
  "size": 10
}

# RESULT: All diseases with code starting with "S82"
# Example: S82.899.A, S82.111.B, S82.456.C
```

### Using Curl

#### Simple Search

```bash
curl -X POST "http://localhost:9200/diseases_v1/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "multi_match": {
        "query": "fever",
        "fields": [
          "diseaseName^6",
          "diseaseCode^8",
          "categoryName^3",
          "subCategoryName^2",
          "healthStandardName"
        ]
      }
    },
    "filter": {
      "term": { "isActive": true }
    },
    "size": 20,
    "from": 0
  }'
```

#### Pagination Curl

```bash
# Page 1
curl -X POST "http://localhost:9200/diseases_v1/_search" \
  -H "Content-Type: application/json" \
  -d '{"size": 20, "from": 0, "query": {"match_all": {}}}'

# Page 2
curl -X POST "http://localhost:9200/diseases_v1/_search" \
  -H "Content-Type: application/json" \
  -d '{"size": 20, "from": 20, "query": {"match_all": {}}}'

# Page 3
curl -X POST "http://localhost:9200/diseases_v1/_search" \
  -H "Content-Type: application/json" \
  -d '{"size": 20, "from": 40, "query": {"match_all": {}}}'
```

#### Index Statistics

```bash
curl -X GET "http://localhost:9200/diseases_v1/_stats"
```

**Response:**

```json
{
  "indices": {
    "diseases_v1": {
      "primaries": {
        "docs": {
          "count": 5000,
          "deleted": 125
        },
        "store": {
          "size_in_bytes": 2048576
        },
        "indexing": {
          "index_total": 5125,
          "index_time_in_millis": 3500
        },
        "search": {
          "query_total": 2341,
          "query_time_in_millis": 4523
        }
      }
    }
  }
}
```

---

## Testing & Validation

### Test 1: Basic Search Functionality

```bash
# Scenario: Search for a common disease
searchKey="fever"

# Expected:
# ✓ Results returned
# ✓ Status 200
# ✓ No duplicates (each ID appears once)
# ✓ Pagination works
```

### Test 2: Pagination Correctness

```bash
searchKey="fever"

# Page 1: documents 0-19
# Page 2: documents 20-39
# Page 3: documents 40-59

# VERIFY:
# ✓ Page 1: 20 unique disease IDs
# ✓ Page 2: 20 DIFFERENT disease IDs (not duplicates of page 1)
# ✓ Page 3: 20 DIFFERENT disease IDs (not duplicates of pages 1-2)
# ✓ Total pages = ceil(totalElements / pageSize)
```

### Test 3: Typo Tolerance

```bash
Tests:
1. searchKey="feavor" (misspelling) → finds "Fever" ✓
2. searchKey="fevr" (missing letter) → finds "Fever" ✓
3. searchKey="feeveer" (extra letters) → finds "Fever" ✓
4. searchKey="fwver" (wrong letter) → finds "Fever" ✓
```

### Test 4: Code Search

```bash
Tests:
1. searchKey="S82" → finds "S82.899.A", "S82.111.B", ... ✓
2. searchKey="s82" (lowercase) → finds same results ✓
3. searchKey="S82899A" (normalized) → finds "S82.899.A" ✓
4. searchKey="s82.899.a" (with dots) → finds "S82.899.A" ✓
```

### Test 5: Input Validation

```bash
Tests:
1. searchKey="" (empty) → returns empty result ✓
2. searchKey=" " (spaces only) → returns empty result ✓
3. searchKey="f" (1 char) → returns empty result ✓
4. searchKey="fe" (2 chars) → returns results ✓
5. searchKey=null → returns empty result ✓
```

### Test 6: Active Filter

```bash
Tests:
1. Create disease with isActive=false
2. Index to Elasticsearch
3. Search for disease by name
4. VERIFY: Disease NOT returned (filter active=true works) ✓
```

### Test 7: Fallback Mechanism

```bash
# Scenario: Elasticsearch is down
# Stop Docker container: docker stop elasticsearch

# Action: Make search request
POST http://localhost:8080/api/v1/user/disease/page
{
  "searchKey": "fever",
  "page": 0,
  "size": 10
}

# Expected:
# ✓ Request succeeds (falls back to SQL)
# ✓ Response time may be slower (SQL vs ES)
# ✓ Results returned (same as before ES)
# ✓ Log shows: "ES search failed, falling back to SQL"
```

### Test 8: Performance Baseline

```bash
Elasticsearch:
- Query latency: 5-50ms
- Throughput: > 1000 queries/second (single node)

SQL (Fallback):
- Query latency: 100-300ms
- Throughput: 100-300 queries/second

Improvement:
- Speed: 2-10x faster with ES
- Throughput: 5-10x better with ES
```

---

## Troubleshooting

### Problem 1: Index Not Created

**Symptom:**

```
WARN: Failed to create Elasticsearch index (Elasticsearch may be unavailable)
```

**Root Cause:**

- Elasticsearch not running
- Connection refused (wrong host/port)
- ES security enabled without credentials

**Solution:**

```bash
# Check ES is running
curl -X GET "http://localhost:9200/_cluster/health"

# If connection refused, verify:
# 1. Docker container is running: docker ps | grep elasticsearch
# 2. Port 9200 is exposed
# 3. ES_JAVA_OPTS are set correctly
```

### Problem 2: Search Returns No Results

**Symptom:**

```json
{
  "content": [],
  "totalElements": 0,
  "totalPages": 0
}
```

**Root Cause:**

- Diseases not indexed yet
- Search term doesn't match any disease
- `isActive` filter excludes all results
- ES query syntax error

**Solution:**

```bash
# Step 1: Verify documents are indexed
curl -X GET "http://localhost:9200/diseases_v1/_count"

# Expected response:
{
  "count": 5000,
  "status": 200
}

# Step 2: If count is 0, reindex
POST http://localhost:8080/admin/disease/reindex

# Step 3: Check a specific disease exists
curl -X GET "http://localhost:9200/diseases_v1/_doc/1"
```

### Problem 3: Duplicate Results in Pagination

**Symptom:**

```
Page 1: [ID 1, ID 1, ID 2, ID 3, ID 1, ...]
Page 2: [ID 1, ID 2, ID 2, ID 4, ...]
```

**Root Cause:**

- Still using SQL search (not ES)
- ES document contains duplicate field values
- Deduplication not applied

**Solution:**

```bash
# Verify using Elasticsearch (not SQL fallback)
# Check application log:

# ✓ LOG: "Routing to Elasticsearch: searchKey=fever"
# ❌ LOG: "Elasticsearch disabled, using SQL fallback"

# If still getting duplicates in ES results, check:
# 1. Each DiseaseDocument has unique @Id
# 2. No multiple documents per disease
# 3. ES pagination parameters (from/size) are correct
```

### Problem 4: Slow Queries

**Symptom:**

```
Search latency: > 500ms
```

**Root Cause:**

- Large result set (high totalElements)
- Complex query with many boosts
- ES cluster under-resourced
- Database still being used (not ES)

**Solution:**

```bash
# Verify using Elasticsearch
grep "Routing to Elasticsearch" application.log

# Check query performance
curl -X POST "http://localhost:9200/diseases_v1/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "query": { "match_all": {} },
    "size": 20,
    "profile": true
  }'

# If using SQL, enable ES:
# app.search.disease.elasticsearch.enabled = true
```

### Problem 5: ES Cluster Down Causes API Failure

**Symptom:**

```
500 Internal Server Error
No results returned
Log shows: "ES search failed"
```

**Root Cause:**

- Elasticsearch is down
- Fallback not working
- Exception not caught properly

**Solution:**

```bash
# Restart Elasticsearch
docker restart elasticsearch

# Verify ES is healthy
curl -X GET "http://localhost:9200/_cluster/health"

# If fallback not working, check:
# 1. try-catch block around ES call
# 2. Feature flag routing logic
# 3. SQL fallback is configured

# Code should have:
try {
    results = diseaseSearchService.search(...);
} catch (Exception e) {
    log.warn("ES failed, falling back to SQL");
    results = diseaseRepository.search(...);  // Fallback
}
```

### Problem 6: Document Fields Are Null

**Symptom:**

```json
{
  "id": 1,
  "diseaseName": "Fever",
  "categoryId": null,
  "categoryName": null
}
```

**Root Cause:**

- Related entities not loaded (lazy loading failed)
- Mapper not handling null relations
- Disease doesn't have related data in DB

**Solution:**

```bash
# Verify disease has relations in database
SELECT d.disease_id, d.disease_name, dc.category_id, dc.name
FROM disease d
LEFT JOIN disease_category dc ON d.category_id = dc.category_id
WHERE d.disease_id = 1;

# If null values are correct (disease has no category), that's OK
# Mapper will store null

# If should have values but don't:
# 1. Check DiseaseRepository uses JOIN FETCH
# 2. Verify mapper gets fully-loaded entity
# 3. Reindex affected documents

curl -X POST "http://localhost:8080/admin/disease/reindex"
```

---

## Production Deployment

### 1. Infrastructure Setup

**Option A: Managed Elasticsearch (Recommended)**

```
Platform: Elasticsearch Cloud, AWS OpenSearch, or Azure Cognitive Search
Shards: 3-5 (depends on data volume)
Replicas: 2-3 (for HA)
Memory: Min 2GB per shard
CPU: 2 cores minimum
Storage: Depends on data volume (typically 3-5x index size)
```

**Option B: Self-Hosted Elasticsearch**

```
Cluster: 3+ nodes minimum
Node roles: master (1-3), data (3+), ingest (optional)
Networking: Private VPC, no public internet
Security: TLS/SSL, authentication, role-based access
Monitoring: Elasticsearch Monitoring + Prometheus/Grafana
Backups: Daily snapshots to S3/cloud storage
```

### 2. Security Configuration

```yaml
spring:
  elasticsearch:
    uris: https://elasticsearch-prod.example.com:9200
    username: ${ELASTIC_USER}
    password: ${ELASTIC_PASSWORD}
    socket-timeout: 30s
    connect-timeout: 10s

# Environment variables (secrets management)
ELASTIC_USER: elasticsearch
ELASTIC_PASSWORD: <strong-password>
```

### 3. Feature Flag Rollout

```yaml
# Phase 1: Canary (10% users)
app:
  search:
    disease:
      elasticsearch:
        enabled: true
        canary-users: 10%  # 10% of users use ES

# Phase 2: Ramp-up (50% users)
        canary-users: 50%

# Phase 3: Full Rollout (100% users)
        canary-users: 100%

# Phase 4: Cleanup (remove feature flag)
        # Always use ES, never fall back
```

### 4. Monitoring & Alerts

```
Metrics to Monitor:
- ES cluster health (green/yellow/red)
- Search query latency (p50, p95, p99)
- Query throughput (queries/second)
- Index size (MB)
- Heap memory usage (%)
- CPU usage (%)
- Disk usage (%)

Alerts:
- Cluster health != green
- Search latency > 500ms
- Error rate > 1%
- Disk usage > 85%
- Memory usage > 90%
- Node failures
```

### 5. Reindexing Strategy

```
Full Reindex:
- Frequency: Nightly (off-peak)
- Duration: 1-2 hours (for 100K diseases)
- Process:
  1. Load all diseases from DB with relations
  2. Flatten to DiseaseDocument
  3. Batch index to Elasticsearch
  4. Verify count matches expected

Zero-Downtime Reindex:
- Create diseases_v2 index (new schema)
- Index all diseases to v2
- Point diseases alias to v2
- Delete diseases_v1
```

---

## Monitoring & Performance

### Key Metrics

```
Query Performance:
- Median latency: < 50ms (target)
- P95 latency: < 200ms (acceptable)
- P99 latency: < 500ms (max)

Throughput:
- Target: 1000+ queries/second
- Single ES node: ~500 queries/second
- 3-node cluster: ~1500-2000 queries/second

Indexing:
- Bulk rate: 5000-10000 documents/second
- Reindex time: < 5 minutes (for 100K documents)
```

### Health Check Dashboard

```
Kibana Dashboard:
1. Cluster Health
   - Status: Green/Yellow/Red
   - Nodes: 3 active
   - Shards: 9 active (3 shards × 3 replicas)

2. Index Health
   - diseases_v1: 5,000 documents
   - Size: ~150 MB
   - Shard size: ~50 MB average
   - Last reindex: 2 hours ago

3. Query Performance
   - Avg latency: 35ms
   - Max latency: 250ms
   - Queries/sec: 450
   - Errors: 0

4. Resource Usage
   - Memory: 65% (2.6 GB / 4 GB)
   - CPU: 25%
   - Disk: 30% (3 TB used / 10 TB available)
```

### Optimization Tips

```
If Queries Are Slow (> 100ms):

1. Add more shards (split data across more nodes)
2. Reduce query complexity (fewer boost fields)
3. Increase heap memory (if available)
4. Use query profiling to identify bottlenecks
5. Check for slow query logs

If Indexing Is Slow:

1. Increase batch size (100 → 500)
2. Increase number of threads
3. Disable replicas during bulk indexing
4. Use refresh interval 30s (instead of 1s)
5. Ensure DB query is performant (JOIN FETCH)

If Memory/Disk Growing:

1. Delete old/archived documents
2. Use index lifecycle management (ILM)
3. Set data retention policy
4. Archive old indices to cheaper storage
5. Monitor per-shard size
```

---

## Quick Reference Checklist

### Local Development

- [ ] Docker Elasticsearch running on :9200
- [ ] application-dev.yml configured
- [ ] Maven dependencies added
- [ ] Java classes created
- [ ] Bulk reindex completed
- [ ] Search test passing

### Before Production

- [ ] Feature flag implemented (`elasticsearch.enabled`)
- [ ] SQL fallback working (tested ES down scenario)
- [ ] Input validation (reject < 2 chars)
- [ ] Deduplication logic in place
- [ ] Logging configured
- [ ] Performance baseline established
- [ ] Monitoring & alerts set up
- [ ] Security configured (TLS, auth)
- [ ] Backup/restore procedure documented
- [ ] Rollback plan ready

### Production Deployment

- [ ] Canary deployment (10% users first)
- [ ] Monitor error rates & latency
- [ ] Ramp up to 50% → 100%
- [ ] Final verification (all tests passing)
- [ ] Alerts active & team notified
- [ ] Documentation updated
- [ ] Support team trained

---

## Next Steps

- Start with [01-ARCHITECTURE.md](./01-ARCHITECTURE.md) for concepts
- Implement code from [02-IMPLEMENTATION.md](./02-IMPLEMENTATION.md)
- Run tests and deployments from this guide (03-OPERATIONS.md)

**Success Criteria:**
✓ Search latency < 50ms
✓ No duplicate results in pagination
✓ Autocomplete working
✓ Typo tolerance working
✓ ES fallback tested and working
✓ Feature flag controls rollout
