# Disease Elasticsearch Search — Design Patterns & Sync Lifecycle

This document explains the best practices for disease search design in Elasticsearch, what fields to index, and which synchronization and reindexing approaches are production-grade.

**Related Documentation:**

- [code-implementation.md](./code-implementation.md) — Complete developer implementation guide with code examples
- [Implementation.md](./Implementation.md) — Executive summary and architecture overview

---

## 1. What Should Be Indexed for Disease Search

Do **not** index everything by default. Store only fields needed for search, display, filtering, or sync.

| Field                   | Index in ES | Rationale                                              |
| ----------------------- | ----------- | ------------------------------------------------------ |
| `id`                    | ✅ Yes      | Primary key for sync and result linking                |
| `diseaseName`           | ✅ Yes      | Main user search field                                 |
| `diseaseCode`           | ✅ Yes      | Exact and prefix ICD-style search                      |
| `diseaseCodeNormalized` | ✅ Yes      | Dot-insensitive code search (`S82899A`)                |
| `diseaseCategoryId`     | Optional    | Useful if UI filters or displays category              |
| `diseaseCategory`       | Optional    | Useful for display and relevance ranking               |
| `diseaseSubCategoryId`  | Optional    | Same reason as category                                |
| `diseaseSubCategory`    | Optional    | Same reason as category                                |
| `healthStandardId`      | Optional    | Useful for filtering or display                        |
| `standardName`          | Optional    | Useful for display and ranking                         |
| `snomedCTCode`          | Optional    | Useful for audit/display or interoperability           |
| `icd11Code`             | Optional    | Useful for future search and display                   |
| `isActive`              | ✅ Yes      | **Critical:** filter so inactive diseases don't appear |
| `isLinkedToHS`          | Optional    | UI/business filter                                     |
| `isSeverityVisible`     | Optional    | UI/business filter                                     |

**Minimum useful index for dropdown search:**

- `id`
- `diseaseName`
- `diseaseCode`
- `diseaseCodeNormalized`
- `isActive`
- Small set of display fields shown in the dropdown

**Anti-pattern:** Do not index every database column by default. Keep Elasticsearch lean.

---

## 2. Search Behavior Recommendations for Production

Use a strict ranking order that prioritizes precision:

1. **Exact `diseaseCode`** — highest confidence
2. **Prefix `diseaseCode`** — likely what user typed
3. **Normalized code prefix (no dots)** — handles `S82899A` style searches
4. **Exact phrase `diseaseName`** — full phrase match
5. **`diseaseName` autocomplete** — keystroke-driven, `edge_ngram`
6. **Typo-tolerant fuzzy match** — `fuzziness: AUTO`
7. **Phonetic match** — weakest fallback for similarly-sounding names

**Critical rule:** Never allow a blank or one-character query to dump the whole disease catalog. Always enforce:

- Minimum query length (≥2 characters)
- Return empty result for blank queries
- Limit result size to prevent dropdown bloat

**For medical search:** Phonetic should never dominate exact code or exact name matches. Medical codes are precise; aliases are secondary.

---

## 3. Recommended Sync Lifecycle

### Insert

1. Save disease in the database (SQL commit).
2. After commit succeeds, call `indexDisease(disease)`.
3. If `isActive=true`, save the Elasticsearch document.
4. If `isActive=false`, ensure it is not searchable (either skip or mark as inactive).

### Update

1. Update disease in the database (SQL commit).
2. After commit succeeds, call `indexDisease(disease)` again.
3. This is an upsert-style sync; no need for separate code paths.

### Delete (Hard)

1. Delete the disease from the database (SQL commit).
2. After commit succeeds, call `deleteDiseaseFromIndex(id)`.
3. The disease is removed from Elasticsearch immediately.

### Deactivate (Soft Delete)

If the project uses `isActive=false` instead of hard delete:

- **Option A (recommended):** Keep the document and filter by `isActive=true` in all queries.
- **Option B:** Remove the document completely when deactivated.

For this project, filtering by `isActive` is already part of the domain, so keeping the document and filtering is cleaner.

### Important: Service Layer Ownership

❌ **Don't:** Call ES sync methods directly from the controller.

✅ **Do:** Keep all sync in the service layer after the DB transaction succeeds, using:

- `@TransactionalEventListener(phase = AFTER_COMMIT)` for simple cases
- Event/domain-event publishing for medium complexity
- Outbox/Kafka for stronger enterprise reliability

This ensures the index is only updated after the database transaction commits.

**For complete code implementation:** See section 8.1 "Incremental Sync: Add/Update/Delete Hooks" in [code-implementation.md](./code-implementation.md) for:

- `DiseaseIndexingService.java` implementation
- How to modify `DiseaseServiceImpl.save()` and `delete()` methods
- Event-driven alternative using `@TransactionalEventListener`

---

## 4. Reindexing Strategy

Manual-only reindex is **not** sufficient for production.

**Recommended production pattern:**

1. **Bootstrap (on startup):** `ApplicationRunner` creates index, mapping, analyzers, and alias if they don't exist
2. **Incremental sync (at runtime):** Every insert/update/delete triggers an index sync
3. **Full reindex (as needed):** Admin endpoint or one-off job for schema changes or data recovery

**Zero-downtime reindex pattern (best for large systems):**

1. Build a new versioned index, e.g., `diseases_v2`
2. Bulk load data into the new index
3. Validate counts and sample searches
4. Atomically switch an alias from `diseases_v1` to `diseases_v2`
5. Keep `diseases_v1` for a rollback window, then retire

---

## 5. ApplicationRunner vs Other Options

| Option                                      | Good For                                                      | Not Good For                      | Recommendation                     |
| ------------------------------------------- | ------------------------------------------------------------- | --------------------------------- | ---------------------------------- |
| `ApplicationRunner`                         | Index bootstrap, mapping creation, alias setup, health checks | Full reindex, heavy startup work  | Use for **bootstrap only**         |
| Admin endpoint                              | On-demand reindex, recovery, operational control              | Automatic sync                    | Use for **full rebuilds**          |
| `@TransactionalEventListener(AFTER_COMMIT)` | Incremental sync after DB write                               | Cross-service durability, replay  | **Good simple choice**             |
| Kafka/Outbox worker                         | Durable async sync, retries, scale                            | Simpler projects with low traffic | **Best for enterprise**            |
| Scheduled reconciliation job                | Periodic drift repair                                         | Real-time sync                    | Use as **safety net**, not primary |

---

## 6. Best Company-Standard Approach

For a company-grade implementation:

**Do this:**

1. `ApplicationRunner` only for index bootstrap and validation.
2. Incremental document sync after DB commit (via `@TransactionalEventListener` or event bus).
3. Admin-triggered full reindex endpoint for recovery and schema changes.
4. Versioned index names (`diseases_v1`, `diseases_v2`) with aliases.
5. Optional: Event-driven sync with Kafka/outbox for stronger reliability.

**Don't do this:**

- ❌ Don't reindex the full disease table on every application start.
- ❌ Don't index every database column by default.
- ❌ Don't let the controller update Elasticsearch directly.
- ❌ Don't expose a search endpoint that returns the full disease catalog on blank input.

---

## 7. Production-Grade Disease Search Flow

**Recommended runtime flow:**

1. Frontend calls `POST /api/v1/user/disease/page` with `searchKey`.
2. Service validates query length and active filter.
3. If Elasticsearch is enabled and healthy, search ES.
4. If ES is unavailable, fall back to the existing database query.
5. Save, update, and delete operations sync the index after commit.
6. An admin-only reindex job or endpoint rebuilds the index when needed.

**Error handling:**

- Log ES search errors, don't throw.
- Fall back to DB silently.
- Monitor fallback rates; alert if consistently > 5%.
- Track ES cluster health (disk space, node count, query latency).

---

## 8. Test Cases To Add or Verify

**Functional tests:**

- ✅ Exact disease code search (e.g., `S82.899.A`)
- ✅ Normalized code search (e.g., `S82899A`)
- ✅ Prefix code search (e.g., `S82`)
- ✅ Disease name search (e.g., `fracture`)
- ✅ Fuzzy typo search (e.g., `frcture` → `fracture`)
- ✅ Phonetic match for similarly-sounding names
- ✅ Empty query handling (should return empty result, not full catalog)
- ✅ One-character query handling (should return empty or very limited result)
- ✅ Inactive disease hidden from results
- ✅ Hard delete removes document from index
- ✅ Update refreshes document in index
- ✅ Elasticsearch down → SQL fallback works

**Operational tests:**

- ✅ Index creation on startup (ApplicationRunner)
- ✅ Reindex of large batch in chunks (memory, throughput)
- ✅ Alias swap for a rebuilt index (atomic, zero-downtime)
- ✅ Search latency under expected load
- ✅ Result size capped to prevent large dropdown payloads

---

## 9. Current Disease Search Flow (Reference)

The disease search starts in `DiseaseController`, goes through `DiseaseServiceImpl.paginate()`, and ends in `DiseaseRepository.getDiseaseLikeAndIsActive()`.

Current SQL-based behavior:

1. Frontend calls `POST /api/v1/user/disease/page`
2. `BaseController.paginate()` delegates to `DiseaseService.paginate()`
3. `DiseaseServiceImpl.paginate()` checks `searchKey`
4. If empty, returns diseases by `isActive` only
5. If present, uses LIKE on name/code + REPLACE() + soundex() fallback
6. Results mapped to `DiseaseDto` and returned

For the **specific disadvantages of this approach and what Elasticsearch solves**, see section 4 in [code-implementation.md](./code-implementation.md).

---

## Final Recommendation

**For a company-grade implementation, use Elasticsearch as a read model only.**

Do this:

- Keep the database as the source of truth.
- Index only the fields needed for search and display.
- Sync the index after DB commit on insert/update/delete.
- Use `ApplicationRunner` only for bootstrap tasks.
- Use a manual/admin reindex path for full rebuilds.
- Prefer versioned indices and aliases for production rebuilds.
- Use Kafka/outbox if the organization wants stronger reliability.

Do not do this:

- Do not reindex the full disease table on every application start.
- Do not index every database column by default.
- Do not let the controller update Elasticsearch directly.
- Do not expose a search endpoint that returns the full disease catalog on blank input.
