# Disease Search: SQL → Elasticsearch Architecture Guide

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Why Elasticsearch](#why-elasticsearch)
3. [Core Concepts](#core-concepts)
4. [Proposed Architecture](#proposed-architecture)
5. [Data Flow Diagrams](#data-flow-diagrams)
6. [Index Design](#index-design)

---

## Problem Statement

### Current Situation (SQL-Based Search)

**Current Flow:**

```
User searches "fever" → SQL query with LEFT JOINs → Multiple database tables
→ One disease matches multiple rows → Duplicate IDs in result set
→ Pagination consumes duplicate slots → User gets fewer unique diseases per page
```

### The Duplicate ID Problem

When searching for diseases, the SQL query JOINs three related tables:

```sql
SELECT d.disease_id, d.disease_name, dc.category_id, dc.category_name, ...
FROM disease d
LEFT JOIN disease_category dc ON d.category_id = dc.category_id
LEFT JOIN disease_subcategory dsc ON d.subcategory_id = dsc.subcategory_id
LEFT JOIN health_standard hs ON d.health_standard_id = hs.health_standard_id
WHERE (d.disease_name LIKE '%fever%' OR dc.name LIKE '%fever%' ...)
LIMIT 20 OFFSET 0;
```

**Result:** One disease appears 3 times (matches through different paths)

- Page 1 slots: IDs [1, 1, 1, 2, 2, 3, 3, 3, 4, 5...] (15 slots = 5 unique diseases + 10 duplicate slots)
- Page 2 starts with more duplicates of diseases from Page 1
- User never discovers better matches on later pages

### Limitations of Current Approach

| Issue                              | Impact                            |
| ---------------------------------- | --------------------------------- |
| `LIKE '%term%'` causes table scans | 100-300ms query latency at scale  |
| No first-class autocomplete        | Users must type full disease name |
| Pagination broken by duplicates    | Users see fewer results per page  |
| Complex SQL tuning                 | Requires DB expertise             |
| No typo tolerance                  | Misspellings return no results    |
| Result ordering embedded in SQL    | Hard to adjust relevance          |

---

## Why Elasticsearch

Elasticsearch is a **purpose-built search engine** with an inverted index architecture that solves all the above problems:

### The Inverted Index (How ES Works)

```
Traditional MySQL Row Storage:
  Row 1: [disease_id, disease_name, category_id, ...]
  Row 2: [disease_id, disease_name, category_id, ...]
  Row 3: [disease_id, disease_name, category_id, ...]
  → To find "fever", scan ALL rows (expensive)

Elasticsearch Inverted Index:
  Token "fever" → [document_id: 1, document_id: 42, document_id: 89]
  Token "infectious" → [document_id: 1, document_id: 3, document_id: 7]
  → To find "fever", jump directly to token (instant)
```

### Elasticsearch Benefits

| Benefit                  | How It Works                                  | Impact                               |
| ------------------------ | --------------------------------------------- | ------------------------------------ |
| **Fast Search**          | Inverted index (pre-built at index time)      | 5-50ms queries (vs 100-300ms SQL)    |
| **Low CPU**              | Token lookup instead of row scan              | Offloads DB load to ES cluster       |
| **Native Autocomplete**  | `edge_ngram` analyzer tokenizes at index time | Type "f" → instant suggestions       |
| **Typo Tolerance**       | `fuzziness: AUTO` at query time               | "feavor" → finds "fever"             |
| **Configurable Ranking** | TF-IDF/BM25 scoring                           | Tune relevance without SQL           |
| **Full-Text Search**     | Phrase, wildcard, phonetic support            | "infectious disease", "WHO", "S82\*" |
| **No Duplicates**        | One document per disease                      | Pagination works correctly           |
| **Scalability**          | Distributed cluster                           | Handles millions of queries/day      |

---

## Core Concepts

### 1. Denormalization (The Key Insight)

**MySQL Approach (NORMALIZED - causes duplicates):**

```
disease (1 row)
├── category (JOIN - separate row)
├── subcategory (JOIN - separate row)
└── health_standard (JOIN - separate row)
Result: 1 disease × 3 table joins = 3 duplicate rows
```

**Elasticsearch Approach (DENORMALIZED - no duplicates):**

```
DiseaseDocument (1 document contains all data)
├── id, diseaseName, diseaseCode (from disease table)
├── categoryId, categoryName (FLATTENED from category table)
├── subCategoryId, subCategoryName (FLATTENED from subcategory table)
└── healthStandardId, healthStandardName (FLATTENED from standard table)
Result: 1 document = 1 disease (no duplicates)
```

**Flattening means:** Copy related-table data INTO the disease document, not reference it

### 2. One Document Per Disease (Single Source of Truth)

```
Rule: 1 Disease Entity = 1 Elasticsearch Document

❌ WRONG:
  - Disease doc references category doc
  - Multiple documents for one disease
  - Join operations in search

✅ CORRECT:
  - Disease doc contains all data needed for search
  - One document, complete information
  - No lookups during search
```

### 3. Mapper Layer (Entity → Document Conversion)

```
MySQL Row:
  disease_id=1, disease_name="Fever", category_id=10, category_name="Infectious"

        ↓ (JPA loads)

JPA Entity (Disease):
  id=1, diseaseName="Fever", diseaseCategory=(id=10, name="Infectious")

        ↓ (Mapper.fromDisease())

ES Document (DiseaseDocument):
  id=1, diseaseName="Fever", categoryId=10, categoryName="Infectious"

        ↓ (elasticsearchOperations.save())

Elasticsearch Index:
  Document _id=1 with all fields stored
```

**Why separate mapper?**

- Entity has `@ManyToOne` lazy relations → can't be indexed directly
- Mapper flattens data → clean denormalized document
- Mapper normalizes codes → "S82.899.A" → "S82899A"
- Keeps concerns separated (ORM ≠ Search)

### 4. Feature Flag (Safe Rollout)

```java
Feature Flag: elasticsearch.enabled

if (elasticsearch.enabled && searchKey.isNotEmpty()) {
    results = searchViaElasticsearch();  // ES
} else {
    results = searchViaDatabase();       // SQL fallback
}

On ES error → Fallback to SQL automatically
```

Enables:

- Canary deployment (enable for 10% users first)
- Zero-downtime rollbacks
- A/B testing search quality

---

## Proposed Architecture

### High-Level System Design

```
┌─────────────────────────────────────────────────────────────┐
│                     USER REQUEST                            │
│              Search "fever" for disease lookup              │
└────────────────────────┬────────────────────────────────────┘
                         ↓
        ┌────────────────────────────────┐
        │    DiseaseController           │
        │   @PostMapping("/page")        │
        │   searchKey="fever"            │
        └────────────────────┬───────────┘
                             ↓
        ┌────────────────────────────────────────┐
        │    DiseaseServiceImpl.paginate()        │
        │  - Check feature flag                  │
        │  - Route to ES or DB                   │
        └────────────┬─────────────┬─────────────┘
                     │             │
        ┌────────────▼──┐   ┌──────▼──────────────┐
        │ ElasticSearch │   │  MySQL (Fallback)  │
        │  DiseaseDoc   │   │  DiseaseRepository │
        │  Search       │   │  Query             │
        └────────────┬──┘   └──────┬──────────────┘
                     │             │
                     └─────┬───────┘
                           ↓
        ┌────────────────────────────────┐
        │  DiseaseDto List               │
        │  - Deduplicated                │
        │  - Paginated                   │
        │  - Scored/Ranked               │
        └────────────────────┬───────────┘
                             ↓
                    ┌────────────────┐
                    │  API Response  │
                    │   (JSON)       │
                    └────────────────┘
```

### Write Path (Index Sync)

```
USER CREATES/UPDATES DISEASE
        ↓
┌──────────────────────────┐
│ DiseaseServiceImpl.save() │
│ 1. Save to MySQL         │
└────────────┬─────────────┘
             ↓
┌─────────────────────────────────────────┐
│ DiseaseIndexingService.indexDisease()   │
│ 1. Load Disease from DB                 │
│ 2. Flatten related data (category, etc) │
│ 3. Save DiseaseDocument to ES           │
└──────────────────┬──────────────────────┘
                   ↓
        ┌──────────────────────┐
        │  Elasticsearch Index │
        │  diseases_v1         │
        └──────────────────────┘
```

### Fallback Strategy

```
┌─ Elasticsearch Call (Try)
│  ├─ Success → Return ES results
│  └─ Timeout/Error → Catch exception
│
└─ Fallback to MySQL (Automatic)
   └─ Return SQL results
   └─ Log error for monitoring
```

---

## Data Flow Diagrams

### Search Flow: "fever" Query Step-by-Step

```
1. QUERY BUILDING
   User input: "fever"
   ├─ Validate: length > 2 ✓
   ├─ Normalize: "fever" (already lowercase)
   └─ Prepare: build BoolQuery

2. ELASTICSEARCH QUERY
   Query: multi_match on [diseaseName, diseaseCode, categoryName]
   Filters: isActive=true
   ├─ Inverted index lookup: "fever" token
   ├─ Matching documents: [1, 42, 89, ...]
   └─ Score each: TF-IDF calculation

3. SCORING (6-Level Boost)
   Document 1 (Fever):
   ├─ diseaseName exact: 12x boost ✓
   ├─ categoryName "Infectious": 2x boost ✓
   └─ Final score: 14 units

   Document 42 (Scarlet Fever):
   ├─ diseaseName phrase: 6x boost ✓
   └─ Final score: 6 units

4. RESULTS RANKED
   [1 (score:14), 42 (score:6), 89 (score:2)]

5. PAGINATION APPLIED
   Page 1: from=0, size=20 → Return documents 1-20
   Page 2: from=20, size=20 → Return documents 21-40
   (NO DUPLICATES - each document returned once)

6. DTO CONVERSION
   DiseaseDocument → DiseaseDto
   Add display names, format data

7. RESPONSE
   {
     "content": [
       { id: 1, name: "Fever", category: "Infectious" },
       { id: 42, name: "Scarlet Fever", category: "Infectious" },
       ...
     ],
     "totalElements": 156,
     "currentPage": 1,
     "totalPages": 8
   }
```

### Indexing Flow: User Creates Disease

```
1. USER ACTION
   Create: Disease(name="COVID-19", code="COVID", categoryId=10)

2. CONTROLLER
   @PostMapping("/disease")
   diseaseService.createDisease(diseaseDto)

3. SERVICE: Save to MySQL
   disease = diseaseRepository.save(disease)
   Result: disease.id = 1001

4. SERVICE: Trigger Indexing
   diseaseIndexingService.indexDisease(1001)

5. MAPPER: Flatten Related Data
   disease = diseaseRepository.findById(1001) // Load with relations
   diseaseDocument = DiseaseSearchDocumentMapper.fromDisease(disease)

   Result DiseaseDocument:
   ├─ id: 1001
   ├─ diseaseName: "COVID-19"
   ├─ diseaseCode: "COVID"
   ├─ diseaseCodeNormalized: "covid"
   ├─ categoryId: 10 (FLATTENED)
   ├─ categoryName: "Infectious Diseases" (FLATTENED)
   └─ isActive: true

6. INDEX TO ELASTICSEARCH
   elasticsearchOperations.save(diseaseDocument)

   PUT /diseases_v1/_doc/1001
   { "id": 1001, "diseaseName": "COVID-19", ... }

7. RESULT
   Document indexed and searchable
```

---

## Index Design

### Document Structure

```java
@Document(indexName = "diseases_v1")
public class DiseaseDocument {
    @Id
    private Long id;                          // Elasticsearch document ID

    // From disease table
    private String diseaseName;               // Text search field
    private String diseaseCode;               // Code search field
    private String diseaseCodeNormalized;     // Normalized code (dots removed)
    private Boolean isActive;                 // Filter field

    // FLATTENED from disease_category table
    private Long categoryId;                  // Filter field
    private String categoryName;              // Boost relevance + display

    // FLATTENED from disease_subcategory table
    private Long subCategoryId;               // Filter field
    private String subCategoryName;           // Boost relevance + display

    // FLATTENED from health_standard table
    private Long healthStandardId;            // Filter field
    private String healthStandardName;        // Boost relevance + display
}
```

### Field Mapping Strategy

| Field                   | Type           | Analyzer                         | Purpose                       |
| ----------------------- | -------------- | -------------------------------- | ----------------------------- |
| `id`                    | long           | -                                | Primary identifier            |
| `diseaseName`           | text           | standard + edge_ngram + phonetic | Autocomplete + typo tolerance |
| `diseaseCode`           | text + keyword | -                                | Exact code search             |
| `diseaseCodeNormalized` | keyword        | -                                | Normalized code (filter)      |
| `categoryId`            | keyword        | -                                | Filtering by category         |
| `categoryName`          | text           | standard                         | Relevance boosting            |
| `subCategoryId`         | keyword        | -                                | Filtering by subcategory      |
| `subCategoryName`       | text           | standard                         | Relevance boosting            |
| `healthStandardId`      | keyword        | -                                | Filtering by standard         |
| `healthStandardName`    | text           | standard                         | Relevance boosting            |
| `isActive`              | boolean        | -                                | Must-filter (active only)     |

### Index Settings

```
Index Name: diseases_v1
Shards: 3
Replicas: 1
Refresh Interval: 1s (faster indexing, acceptable latency)

Analyzers:
  - standard (default): lowercase + tokenize
  - edge_ngram: f, fe, fev, feve, fever (autocomplete)
  - phonetic: fever, feavor, fevr (typo tolerance)
```

### Ranking Strategy (6-Level Boost)

```
Search for "fever":

Level 1 (EXACT CODE MATCH): boost 12x
  Example: code="FVR" matches "FVR" exactly

Level 2 (CODE PREFIX): boost 8x
  Example: code="FEVER*" matches "FEVER", "FEVER-A"

Level 3 (PHRASE MATCH): boost 6x
  Example: diseaseName="Fever" contains "fever"

Level 4 (AUTOCOMPLETE): boost 4x
  Example: diseaseName="f|fe|fev|feve|fever" matches edge_ngram

Level 5 (FUZZY MATCH): boost 2x
  Example: "feavor" fuzzy-matches "fever"

Level 6 (PHONETIC FALLBACK): boost 1x
  Example: "feavor" phonetically sounds like "fever"

Result Ranking:
  Document with exact code match: 12 points
  Document with phrase match: 6 points
  Document with fuzzy match: 2 points
  → Returned in score order
```

---

## Key Principles

### 1. No Lazy Loading in Search Documents

```
❌ WRONG:
@Document
public class Disease {
    @ManyToOne(fetch = LAZY)  // NEVER in ES document
    private DiseaseCategory category;
}

✅ CORRECT:
public class DiseaseDocument {
    private Long categoryId;      // Flattened
    private String categoryName;  // Flattened
}
```

### 2. One Document = One Disease

```
❌ WRONG: Multiple documents per disease (one per category)
✅ CORRECT: Single document with all category/subcategory data flattened
```

### 3. Always Filter Active

```
✅ MANDATORY:
Every search query must include: filter: { term: { isActive: true } }
```

### 4. Fallback on Elasticsearch Error

```
✅ MANDATORY:
try {
    return searchViaElasticsearch(query);
} catch (Exception e) {
    log.error("ES failed, falling back to DB", e);
    return searchViaDatabase(query);  // Automatic fallback
}
```

### 5. Feature Flag Controls Everything

```
✅ MANDATORY:
app.search.disease.elasticsearch.enabled = true/false
Controls: ES enabled or always use DB
```

---

## Summary: Why This Architecture Works

| Problem                 | Solution                              | Result                     |
| ----------------------- | ------------------------------------- | -------------------------- |
| Duplicates from JOINs   | One denormalized document per disease | Pagination works correctly |
| Slow LIKE queries       | Inverted index                        | 5-50ms latency             |
| No autocomplete support | edge_ngram analyzer                   | Type "f" → instant results |
| Typo-prone search       | Fuzzy + phonetic matching             | "feavor" finds "fever"     |
| Hard to tune ranking    | TF-IDF scoring in query DSL           | Adjust without SQL changes |
| Scalability limits      | Distributed Elasticsearch cluster     | Millions of queries/day    |
| No fallback option      | Feature flag + SQL fallback           | Safe canary deployment     |

---

## Next Steps

→ **[Read 02-IMPLEMENTATION.md](./02-IMPLEMENTATION.md)** for complete code and setup instructions
