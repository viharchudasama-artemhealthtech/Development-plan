# Elasticsearch Integration for Disease/Medicine Search

## Overview

This project migrates search from traditional SQL `LIKE` queries to Elasticsearch for disease and medicine lookup. The goal is to make search faster, more flexible, and more user-friendly as the dataset grows to around 70,000 indexed records.

SQL `LIKE` queries are simple, but they become slower and less relevant as search requirements grow. Elasticsearch is a better fit for this use case because it supports:

- Full-text search with relevance scoring
- Fuzzy search for misspellings
- Autocomplete for fast type-ahead suggestions
- Phonetic search for similar-sounding terms
- Better search performance at scale

This makes the search experience more responsive for users while keeping the backend ready for production growth.

## Tech Stack

- Spring Boot
- Spring Data Elasticsearch
- Elasticsearch
- MySQL / PostgreSQL
- Java

## Architecture / Flow

```text
Database -> Spring Boot -> Elasticsearch -> API -> Frontend
```

### How it works

1. Source data is stored in MySQL or PostgreSQL.
2. Spring Boot reads the source records and transforms them into Elasticsearch documents.
3. Elasticsearch indexes the data for fast search.
4. The API queries Elasticsearch instead of running expensive SQL `LIKE` searches.
5. The frontend receives faster, more relevant results.

This architecture also supports a fallback strategy, so the application can return database search results if Elasticsearch is unavailable.

## Elasticsearch Index Design

A lean Elasticsearch document should contain only the fields needed for search and filtering.

### Sample document structure

```json
{
  "id": "101",
  "name": "Paracetamol 500 mg",
  "code": "PCM500",
  "isActive": true
}
```

### Important fields

- `name`: searchable medicine or disease name
- `code`: medicine code, ICD code, or internal reference code
- `isActive`: enables filtering of only active records

Keeping the document minimal reduces index size and improves query speed.

## Indexing Strategies

There are multiple ways to load data into Elasticsearch depending on the application lifecycle and sync requirements.

### 1. Bulk Indexing (Reindex API)

This is the most common and safest way to build the index when a large number of records already exist in the database. The application reads the source table in pages, maps each row into a lean Elasticsearch document, and writes the documents in batches. For a dataset of around 70,000 records, this approach is usually the best starting point because it is easy to control, easy to monitor, and simple to restart if a batch fails.

Why it works well:

- It prevents memory pressure by processing records in chunks.
- It allows progress tracking and retry handling.
- It is easy to expose as an admin-only endpoint.
- It fits one-time migrations and full reindex operations.

```java
@PostMapping("/admin/reindex")
public ResponseEntity<String> reindex() {
    indexingService.reindexAll();
    return ResponseEntity.ok("Reindex started");
}
```

Use this when:

- the index is empty
- mappings have changed
- a full refresh is required after deployment
- you need a predictable migration path for existing production data

Operational note:

- Keep batch sizes in the 500 to 1000 range.
- Temporarily lower refresh frequency during large imports.
- Log each batch so failures can be retried without repeating the entire job.

### 2. Elasticsearch Bulk API

Bulk API is the fastest ingestion method when the goal is to push many documents quickly into Elasticsearch. Instead of sending one request per record, the application sends many index operations in a single bulk request. This reduces network overhead, improves throughput, and is much more efficient for large data loads.

Compared with plain repository save calls, Bulk API gives you better control over:

- request size
- partial failure handling
- throughput tuning
- indexing performance under load

```java
List<IndexQuery> queries = diseases.stream()
    .map(disease -> new IndexQueryBuilder()
        .withId(disease.getId().toString())
        .withObject(diseaseMapper.toDocument(disease))
        .build())
    .toList();

elasticsearchOperations.bulkIndex(queries, IndexCoordinates.of("disease_search"));
```

Use this when processing thousands of records in batches, especially if reindexing is part of a larger ETL-style workflow.

Best fit:

- large one-time migrations
- backfill jobs
- high-volume sync jobs

Tradeoff:

- It is faster than simple save calls, but it requires more careful error handling and chunk management.

### 3. CommandLineRunner (Startup Indexing)

This approach runs indexing automatically when the application starts. It is convenient because the application can initialize Elasticsearch data without requiring a manual trigger. That makes it useful for local development, demos, or controlled test environments where the index should always be populated after startup.

However, startup indexing can be risky in production because it may slow down application boot, duplicate work after restarts, or trigger an expensive full reindex unintentionally.

```java
@Bean
CommandLineRunner seedIndex(IndexingService indexingService) {
    return args -> indexingService.reindexAll();
}
```

Use this only in development or controlled environments.

Recommended safeguards:

- Guard it with a feature flag.
- Disable it in production profiles.
- Make sure the method is idempotent if it can run more than once.

### 4. Scheduled Cron Job

A scheduled cron job keeps Elasticsearch refreshed at regular intervals. This is useful when the source database changes throughout the day and you want periodic sync without implementing event-driven updates yet. It is a practical middle ground between full manual reindexing and real-time dual write.

The downside is that search data can be slightly stale between runs, so it is better for systems where near-real-time updates are not critical.

```java
@Scheduled(cron = "0 0 2 * * *")
public void scheduledReindex() {
    indexingService.reindexAll();
}
```

Use this when source data changes periodically and near-real-time sync is not required.

Good for:

- nightly refresh jobs
- low-change master data
- backup sync when event streaming is not available

Production note:

- Avoid running a full cron-based reindex too frequently.
- Prefer incremental sync if the dataset changes often.

### 5. Dual Write (Real-Time Sync)

Dual write updates both the database and Elasticsearch whenever a record changes. In this model, the application writes to the database as the source of truth and immediately mirrors the same change into Elasticsearch. This gives users near-real-time search results and keeps the index fresh after create, update, or delete operations.

This approach provides the best user experience for live search, but it also adds operational complexity. If Elasticsearch write fails after the database write succeeds, the application must decide whether to retry, queue the event, or log the failure for later reconciliation.

```java
@Transactional
public DiseaseDto saveDisease(DiseaseDto dto) {
    Disease disease = diseaseRepository.save(mapper.toEntity(dto));
    indexingService.indexDisease(disease.getId());
    return mapper.toDto(disease);
}
```

Use this for near-real-time synchronization. In production, make sure indexing failures do not break the primary transaction flow.

Best practice:

- Keep the database write as the primary transaction.
- Make Elasticsearch updates asynchronous when possible.
- Add retry logic or an outbox pattern for stronger reliability.

## Recommended Strategy

| Step              | Approach               | Why                                                             |
| ----------------- | ---------------------- | --------------------------------------------------------------- |
| Initial indexing  | Bulk API / Reindex API | Best for loading the existing 70,000 records quickly and safely |
| Real-time updates | Dual Write             | Keeps new and changed records searchable immediately            |
| Backup sync       | Scheduled Job          | Helps recover from missed updates or sync drift                 |

In most production systems, the best combination is to use bulk reindexing first, then dual write for live updates, and keep a scheduled job as a safety net.

## How to Run Indexing

### API endpoint

```http
POST /admin/reindex
```

### Steps to execute

1. Start the Spring Boot application.
2. Make sure Elasticsearch is running and reachable.
3. Call the reindex endpoint.
4. Wait for the bulk indexing job to complete.
5. Verify document count and sample searches in Elasticsearch.

Example using `curl`:

```bash
curl -X POST http://localhost:8080/admin/reindex
```

## Search Features

### Fuzzy search

Fuzzy search helps match misspelled or partially correct terms.

Example:

- `paracetmol` can still match `Paracetamol`

### Autocomplete

Autocomplete provides instant suggestions as the user types.

Example:

- Input: `para`
- Output: `Paracetamol`, `Paracetamol 500 mg`

### Phonetic search

Phonetic search helps find words that sound similar.

Example:

- `amoxycillin` and `amoxicillin`

These features significantly improve usability for medical search, where spelling variations are common.

## Performance Optimizations

### Batch processing

Index data in batches instead of one record at a time. This reduces memory pressure, avoids oversized requests, and improves throughput. For this project size, batches of 500 to 1000 records usually offer a good balance between speed and resource usage.

### Disable refresh interval

During large reindex jobs, temporarily disable or increase the refresh interval to improve indexing speed. Elasticsearch refreshes make new documents visible for search, but they also add overhead during heavy ingestion. Reducing refresh frequency during backfills can significantly improve indexing performance.

```json
PUT /disease_search/_settings
{
  "index": {
    "refresh_interval": "-1"
  }
}
```

After indexing completes, restore the refresh interval.

### Use pagination

Always paginate search responses to reduce payload size and improve response time. Pagination also prevents accidental retrieval of too many documents in one response and keeps both the API and frontend responsive.

```java
Pageable pageable = PageRequest.of(page, size);
```

## Best Practices

- Use an index alias such as `disease_search` instead of pointing the app to a versioned index directly.
- Keep only the fields required for search and filtering.
- Use fallback database search when Elasticsearch is unavailable.
- Normalize codes before indexing so search behavior stays consistent.
- Keep indexing logic separate from search logic.

## Common Mistakes

- Indexing all data at once without batching
- No batching during bulk reindexing
- Poor mapping design with too many unused fields
- Querying Elasticsearch without pagination
- Forgetting to filter inactive records
- Skipping fallback DB search during outages

## Future Improvements

- Kafka or CDC integration for event-driven sync
- Advanced ranking and query boosting
- Search analytics and usage insights
- Better synonym handling for medical terminology
- Multi-language support for medical search terms

## Conclusion

Elasticsearch provides a strong upgrade over traditional SQL `LIKE` search for disease and medicine lookup. It delivers faster response times, better relevance, fuzzy matching, autocomplete, and scalable indexing for large datasets.

For a production system, the best pattern is to combine Elasticsearch search, careful batching, lean document design, and a reliable database fallback. This gives users a faster search experience while keeping the system stable and maintainable.
