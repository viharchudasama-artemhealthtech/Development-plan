# Implementation Disease Search — SQL → Elasticsearch Migration

**Related Documentation:**

- [code-implementation.md](./code-implementation.md) — Complete developer guide with step-by-step code changes
- [disease-elasticsearch-search-design copy.md](./disease-elasticsearch-search-design%20copy.md) — Design patterns and sync lifecycle

---

## Executive Summary

**Purpose:** Provide a high-level business and architectural overview of moving Disease Search from SQL to Elasticsearch.

**Business Impact:**

- **Performance:** Lower and more consistent latency for autocomplete and lookup requests; reduced DB CPU from `LIKE` queries
- **Scalability:** Handle large disease catalogs and high query throughput by offloading read-heavy search traffic to ES clusters
- **User Experience:** Instant autocomplete, typo-tolerance, and relevance-ranked suggestions
- **Cost Efficiency:** Move search load from primary database to distributed search cluster

---

> Executive callout: Prefer a debounced autocomplete search bar (top 8–12 suggestions) for day-to-day lookup, and surface a `View all results` action that opens the paginated results view. Keep server-side pagination for full browse/admin workflows to preserve scalability and reproducible paging.

## Current Search Architecture

**Primary files:**

- `src/main/java/.../controller/BaseController.java` — generic `paginate` endpoint
- `src/main/java/.../controller/DiseaseController.java` — disease API layer
- `src/main/java/.../service/impl/DiseaseServiceImpl.java` — `paginate()` business logic
- `src/main/java/.../repository/DiseaseRepository.java` — native SQL queries with `LIKE`, `REPLACE()`, `soundex()`
- `bmc-medico-models/src/main/java/.../model/Disease.java` — JPA entity

**Current flow:**

1. Frontend calls `POST /api/v1/user/disease/page`
2. `BaseController.paginate()` → `DiseaseServiceImpl.paginate()`
3. Service normalizes search key and calls `DiseaseRepository.getDiseaseLikeAndIsActive(...)`
4. Database executes complex SQL with multiple `OR` conditions, `LIKE` patterns, and `soundex()` fallback
5. Results mapped to `DiseaseDto` and returned

**Current limitations:**

- `LIKE '%term%'` causes table scans at scale
- No first-class autocomplete support
- Pagination limitation: misses results outside current database page
- Complex SQL tuning requires database expertise
- No synonym or phonetic flexibility
- Result ordering embedded in SQL `CASE` statements

---

## Why Elasticsearch

Elasticsearch is purpose-built for full-text search with an inverted index architecture:

- **Orders of magnitude faster:** Token postings instead of row scans
- **Lower CPU:** Pre-computed index at insert time, not per-query
- **Native autocomplete:** `edge_ngram` analyzer for keystroke-driven prefix matching
- **Fuzzy matching:** `fuzziness: AUTO` for typo tolerance at query time
- **Configurable:** Analyzers, synonyms, phonetic variants without SQL changes
- **Consistent ranking:** TF-IDF/BM25 scoring across all queries
- **Full-index search:** Not limited to current database page
- **Scalability:** Distributed index cluster handles high query throughput

---

## Proposed Architecture

```
User → Frontend → Backend API (/api/v1/user/disease/page)
   ↓
   ├─ Feature Flag: elasticsearch.enabled
   ├─ YES → DiseaseSearchService → Elasticsearch
   └─ NO or ES Down → DiseaseRepository (SQL fallback)
   ↓
Map results → ResponseDto → Frontend
```

**Key principles:**

- Database remains the source of truth for all writes
- Elasticsearch is a read-only search projection
- Feature flag routes traffic for safe canary deployments
- SQL fallback ensures graceful degradation
- Versioned indices (`diseases_v1`) with aliases for zero-downtime reindexing

---

## Elasticsearch Index Design (Summary)

**Recommended fields:**

- `id` (long) — primary key
- `diseaseName` (text with `edge_ngram` analyzer) — autocomplete and phrase
- `diseaseName.phonetic` (text with phonetic analyzer) — fallback
- `diseaseCode` (keyword + text) — exact and prefix match
- `diseaseCodeNormalized` (keyword) — dot-free code for `S82899A` style search
- `diseaseCategory`, `diseaseSubCategory` (keyword) — filtering and display
- `isActive` (boolean) — filter out inactive diseases

**Ranking strategy (boost weights):**

1. Exact code match: 12x
2. Code prefix: 8x
3. Phrase match on name: 6x
4. Autocomplete (edge_ngram): 4x
5. Fuzzy typo match: 2x
6. Phonetic fallback: 1x

**Alias pattern:**

- Build versioned index: `diseases_v1`, `diseases_v2`, etc.
- Point alias `diseases` to active version
- Enables atomic index swaps for zero-downtime schema updates

---

## Spring Boot Integration

**Dependency** (add to `pom.xml`)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

**Configuration:**

- Create `ElasticsearchConfig.java` with client setup and repository scanning
- Enable `@EnableElasticsearchRepositories`
- Read connection details from `spring.elasticsearch.uris`, authentication, etc.

**Production-Grade Index Setup (Essential):**

- Create `DiseaseSearchIndexConfig.java` — runs on Spring Boot startup and:
  - Creates the `diseases_v1` index with proper shard/replica settings
  - Defines custom analyzers (`edge_ngram` for autocomplete, `phonetic_analyzer` for typo tolerance)
  - Applies field mappings from the `Disease` entity
  - Gracefully degrades if Elasticsearch is unavailable (logs warning, continues with SQL fallback)
- This file is **critical for production** — without it, autocomplete and typo-tolerant search won't work.

**Repositories:**

- Keep `DiseaseRepository` for DB writes
- Add `DiseaseSearchRepository extends ElasticsearchRepository<Disease, Long>` for ES reads

**Service routing:**

- `DiseaseServiceImpl.paginate()` checks feature flag
- If enabled and search key present, call `DiseaseSearchService`
- On ES error, fall back to `DiseaseRepository`

---

## Data Indexing Strategy

**Initial bulk load:**

- Batch reindex from MySQL into Elasticsearch
- Use pagination and bulk operations for efficiency
- Normalize code fields (remove dots) during indexing

**Ongoing incremental sync:**

- After successful DB commit, trigger ES document index/update/delete
- Use `@TransactionalEventListener(phase = AFTER_COMMIT)` or event-driven Kafka
- Recommended for production reliability

**Optional: admin reindex endpoint**

- `POST /admin/disease/reindex` for manual full rebuilds
- Useful for index schema changes
- Keep async to avoid blocking API requests

---

## Testing & Validation

**Functional test cases:**

- Exact disease code search (e.g., `S82.899.A`)
- Normalized code search (e.g., `S82899A`)
- Prefix code search (e.g., `S82`)
- Disease name search (e.g., `fracture`)
- Fuzzy typo search (e.g., `frcture` → `fracture`)
- Phonetic match for similar sounding names
- Empty query handling
- Inactive disease hidden from results
- ES down → SQL fallback works

**Performance validation:**

- Query latency comparison: SQL vs ES
- Autocomplete responsiveness (sub-100ms target)
- Pagination performance
- Index size and memory usage
- Bulk reindex throughput

---

## Operational Risks & Mitigation

| Risk                                    | Mitigation                                                                               |
| --------------------------------------- | ---------------------------------------------------------------------------------------- |
| ES cluster unavailable                  | SQL fallback; monitor ES health; alert on errors                                         |
| Stale ES index                          | Incremental sync on commit; periodic reconciliation; versioned indices                   |
| Version mismatch between index and code | Use versioned index names; alias-based rollover; runbook for emergency reindex           |
| High cost for large clusters            | Start with single node for dev/test; right-size for prod traffic; monitor query patterns |
| Search behavior regression              | A/B test in production; compare result rankings; monitor fallback rates                  |

---

## Rollback Plan

**If ES causes issues:**

1. Disable feature flag `elasticsearch.enabled = false`
2. All `/page` searches route to SQL via `DiseaseRepository`
3. API response shape unchanged; clients unaffected
4. Keep ES index in place until confirmed stable
5. Remove ES dependency only after full rollback verification

---

## For Detailed Technical Implementation

See [code-implementation.md](./code-implementation.md) for:

- Step-by-step code changes with file paths and line numbers
- Disadvantages of current SQL approach (7 issues)
- What Elasticsearch solves (8 benefits)
- Migration plan with 4 phases
- Before/after code examples
- Data indexing scripts
- Testing examples
- Performance impact analysis

See [disease-elasticsearch-search-design copy.md](./disease-elasticsearch-search-design%20copy.md) for:

- Design decisions and patterns
- What to index and why
- Sync lifecycle (insert, update, delete, reindexing)
- Production-grade deployment options
- Test cases and operational guidance

Local quickstart:

```bash
docker run --name es -p 9200:9200 -e discovery.type=single-node docker.elastic.co/elasticsearch/elasticsearch:8.11.0
```

Start app with:

```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
app:
  search:
    disease:
      elasticsearch:
        enabled: true
```

POST `/api/v1/user/disease/page` with body:

```json
{ "searchKey": "fracture", "page": 0, "size": 10 }
```

Verify ranking, sample top-5 results, and latency distribution.

---

## Performance Impact (code perspective)

- SQL `LIKE` scanning → high DB cost and unpredictable latency.
- ES inverted index → token posting lists, highly efficient prefix and fuzzy lookups.
- Expect lower median latency for autocomplete (depends on infra): often tens of ms vs 100–300ms+ for DB-based contains queries. DB CPU for search endpoints should drop notably.

---

## Implementation Roadmap (summary)

Phase 1 — Provision ES (dev/staging/prod), secure cluster.

Phase 2 — Implement mappings and seed `diseases_v1` index.

Phase 3 — Bulk reindex active diseases.

Phase 4 — Deploy ES service (staging) behind `app.search.disease.elasticsearch.enabled=true` and validate.

Phase 5 — Tune analyzers, boosts, page size and monitor.

Phase 6 — Roll out to production and monitor / reconcile.

---

## Risks and Mitigation

- Data inconsistency: mitigate with outbox/event-driven indexing and reconciliation jobs.
- Infra overhead: use managed ES or appropriately sized clusters; ILM and monitoring.
- Relevance regressions: implement relevance tests (query → expected top matches) and CI checks.

---

## Future Enhancements

- Vector/semantic search (embeddings) for concept-aware matching.
- AI-driven ranking and personalization based on selection logs.
- Synonym and abbreviation maps (e.g., `PCM` → `Paracetamol`).

---

## Conclusion

Elasticsearch upgrades the disease search experience, improves performance and scalability, and provides a platform for richer capabilities (typo tolerance, phonetic, semantic search). Implement incrementally behind a feature flag, using bulk + incremental indexing and a reliable outbox for production.

Next steps I can implement for you now:

- (A) add `spring-boot-starter-data-elasticsearch` to `pom.xml`.
- (B) add `DiseaseSearchDocument`, repository, service, and index config drafts to the codebase.
- (C) patch `DiseaseServiceImpl` to call the ES service behind the feature flag and add an after-commit listener.

Reply with option(s) to apply and I will implement them, run local verification, and present results.
