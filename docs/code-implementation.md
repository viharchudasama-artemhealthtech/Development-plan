# Code Implementation README: Disease Search Migration from SQL to Elasticsearch

This document is a developer implementation guide and code migration reference for moving Disease Search in `bmc-user-api` from SQL-based search to Elasticsearch-based search.

Current code references in this guide are based on the workspace snapshot in this session. They point to the exact files and line areas that currently control search behavior.

---

## 1. Objective

The objective of this migration is to replace the current SQL search path for Disease lookup with Elasticsearch-backed search, while keeping the API contract stable for consumers.

This guide explains:

- which files are modified
- what code exists today
- what exact changes are required
- where each change belongs at file and line level
- how to index data, test the search, and roll back safely if needed

### Files involved

Current files to modify:

- [bmc-user-api/pom.xml](../pom.xml)
- [bmc-medico-models/src/main/java/com/hospisoft/model/Disease.java](../../bmc-medico-models/src/main/java/com/hospisoft/model/Disease.java)
- [bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/repository/DiseaseRepository.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/repository/DiseaseRepository.java)
- [bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/DiseaseService.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/DiseaseService.java)
- [bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseServiceImpl.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseServiceImpl.java)
- [bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/controller/DiseaseController.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/controller/DiseaseController.java)

New files to add:

- [bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/config/ElasticsearchConfig.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/config/ElasticsearchConfig.java)
- [bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/repository/DiseaseSearchRepository.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/repository/DiseaseSearchRepository.java)
- [bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/DiseaseSearchService.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/DiseaseSearchService.java)
- [bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseSearchServiceImpl.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseSearchServiceImpl.java)
- [bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/DiseaseIndexingService.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/DiseaseIndexingService.java)
- [bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseReindexService.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseReindexService.java)

---

## 2. Project Structure Overview

The relevant backend structure for this migration is:

```text
bmc-user-api/
└── src/
    └── main/
        └── java/
            └── com/artemhealthtech/bmc/medicodb/api/v1/user/
                ├── controller/
                │   └── DiseaseController.java
                ├── service/
                │   ├── DiseaseService.java
                │   ├── DiseaseSearchService.java          (new)
                │   └── DiseaseIndexingService.java        (new)
                ├── service/impl/
                │   ├── DiseaseServiceImpl.java
                │   ├── DiseaseSearchServiceImpl.java      (new)
                │   └── DiseaseReindexService.java         (new)
                ├── repository/
                │   ├── DiseaseRepository.java
                │   └── DiseaseSearchRepository.java       (new)
                ├── config/
                │   ├── DiseaseSearchIndexConfig.java      (new) ← Index setup, analyzers, mappings
                │   └── ElasticsearchConfig.java           (new)
                └── dto/
                    └── DiseaseDto.java

bmc-medico-models/
└── src/
    └── main/
        └── java/
            └── com/hospisoft/model/
                └── Disease.java
```

The search migration should keep the controller thin, move search execution into a dedicated search service, and keep DB writes in the existing disease service.

---

## 2.1: Production-Grade Index Configuration (Analyzers & Mappings)

**File to add:** [DiseaseSearchIndexConfig.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/config/DiseaseSearchIndexConfig.java)

This configuration file is **essential for production**. It runs automatically on Spring Boot startup via `@Bean ApplicationRunner` and:

1. **Creates the Elasticsearch index** (`diseases_v1`) if it doesn't exist.
2. **Defines custom analyzers** for search quality:
   - `disease_autocomplete` (index-time): applies `edge_ngram` filter (1–20 character n-grams) for keystroke-driven autocomplete.
   - `autocomplete_search` (query-time): simple lowercase for query normalization.
   - `phonetic_analyzer`: applies phonetic encoding (Double Metaphone) for typo-tolerant fallback.
3. **Configures index settings**: number of shards, replicas, max_ngram_diff for edge_ngram safety.
4. **Applies field mappings** from the `Disease` entity (via `@Document` and `@Field` annotations).
5. **Gracefully degrades** if Elasticsearch is unavailable at startup—logs a warning and lets the app boot with SQL fallback.

**Why this is critical:**

- Without `edge_ngram`, autocomplete won't work; queries won't match partial text.
- Without `phonetic_analyzer`, typo-tolerant search won't work.
- Without explicit index settings, Elasticsearch may reject edge_ngram definitions.
- Allows safe reindexing: create `diseases_v2`, switch alias atomically, keep v1 for rollback.

**Location in code:** `src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/config/DiseaseSearchIndexConfig.java`

**Implementation code:**

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.config;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.IndexOperations;
import org.springframework.data.elasticsearch.core.index.Settings;
import org.springframework.data.elasticsearch.core.mapping.IndexCoordinates;

import com.hospisoft.model.Disease;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Production-Grade Elasticsearch Index Configuration.
 *
 * Creates diseases_v1 index on Spring Boot startup with:
 * - Edge n-gram analyzer for keystroke autocomplete (1-20 char n-grams)
 * - Phonetic analyzer for typo-tolerant fallback (Double Metaphone)
 * - Custom field mappings from Disease entity
 * - Graceful degradation if Elasticsearch unavailable
 *
 * Why production-grade:
 * - Defines analyzers explicitly (can't search without them)
 * - Handles zero-downtime reindexing (versioned indices + aliases)
 * - Logs all operations for debugging
 * - Doesn't block app startup if ES fails
 */
@Configuration
@RequiredArgsConstructor
@Slf4j
public class DiseaseSearchIndexConfig {

    private static final String DISEASES_INDEX_V1 = "diseases_v1";
    private final ElasticsearchOperations elasticsearchOperations;

    /**
     * Initialize Elasticsearch index with analyzers and mappings.
     * Runs automatically on Spring Boot startup.
     */
    @Bean
    CommandLineRunner diseaseIndexInitializer() {
        return args -> {
            try {
                IndexOperations indexOps = elasticsearchOperations.indexOps(
                        IndexCoordinates.of(DISEASES_INDEX_V1));

                // Check if index already exists
                if (indexOps.exists()) {
                    log.info("Index {} already exists. Skipping creation.", DISEASES_INDEX_V1);
                    return;
                }

                log.info("Creating Elasticsearch index: {}", DISEASES_INDEX_V1);

                // ============================================================
                // Define Edge N-Gram Filter for Autocomplete
                // ============================================================
                // Edge n-gram generates partial tokens on query:
                // "disease" → ["d", "di", "dis", "dise", "disea", "disease", ...]
                // Allows keystroke-by-keystroke matching (autocomplete)
                Map<String, Object> edgeNgramFilter = new LinkedHashMap<>();
                edgeNgramFilter.put("type", "edge_ngram");
                edgeNgramFilter.put("min_gram", 1);    // Start from 1 char
                edgeNgramFilter.put("max_gram", 20);   // Up to 20 chars
                edgeNgramFilter.put("token_chars", new String[]{"letter", "digit"});

                // ============================================================
                // Define Phonetic Filter for Typo Tolerance
                // ============================================================
                // Double Metaphone encoding:
                // "colour" → encoded similarly to "color" (phonetically similar)
                // Allows typo-tolerant search (e.g., misspelled disease names)
                Map<String, Object> phoneticFilter = new LinkedHashMap<>();
                phoneticFilter.put("type", "phonetic");
                phoneticFilter.put("encoder", "double_metaphone");
                phoneticFilter.put("replace", false);   // Keep original + phonetic tokens

                // ============================================================
                // Define Custom Analyzers
                // ============================================================

                // Analyzer 1: disease_autocomplete (used at INDEX time)
                // Tokenizes with standard tokenizer, applies lowercase + edge_ngram
                Map<String, Object> autocompleteIndexAnalyzer = new LinkedHashMap<>();
                autocompleteIndexAnalyzer.put("type", "custom");
                autocompleteIndexAnalyzer.put("tokenizer", "standard");
                autocompleteIndexAnalyzer.put("filter", new String[]{"lowercase", "autocomplete_filter"});

                // Analyzer 2: autocomplete_search (used at QUERY time)
                // Just lowercase for query normalization (matches edge_ngram index-time)
                Map<String, Object> autocompleteSearchAnalyzer = new LinkedHashMap<>();
                autocompleteSearchAnalyzer.put("type", "custom");
                autocompleteSearchAnalyzer.put("tokenizer", "standard");
                autocompleteSearchAnalyzer.put("filter", new String[]{"lowercase"});

                // Analyzer 3: phonetic_analyzer (fallback for typo tolerance)
                // Applies phonetic encoding for accent/sound-alike matching
                Map<String, Object> phoneticAnalyzer = new LinkedHashMap<>();
                phoneticAnalyzer.put("type", "custom");
                phoneticAnalyzer.put("tokenizer", "standard");
                phoneticAnalyzer.put("filter", new String[]{"lowercase", "phonetic_filter"});

                // ============================================================
                // Assemble Analysis Configuration
                // ============================================================
                Map<String, Object> analysis = new LinkedHashMap<>();

                Map<String, Object> filters = new LinkedHashMap<>();
                filters.put("autocomplete_filter", edgeNgramFilter);
                filters.put("phonetic_filter", phoneticFilter);
                analysis.put("filter", filters);

                Map<String, Object> analyzers = new LinkedHashMap<>();
                analyzers.put("disease_autocomplete", autocompleteIndexAnalyzer);
                analyzers.put("autocomplete_search", autocompleteSearchAnalyzer);
                analyzers.put("phonetic_analyzer", phoneticAnalyzer);
                analysis.put("analyzer", analyzers);

                // ============================================================
                // Create Index Settings
                // ============================================================
                // max_ngram_diff=19 allows edge_ngram(min=1, max=20)
                Map<String, Object> indexSettings = new LinkedHashMap<>();
                indexSettings.put("number_of_shards", 1);      // Single shard for small data
                indexSettings.put("number_of_replicas", 0);    // No replicas (can scale later)
                indexSettings.put("max_ngram_diff", 19);       // Allow 1-20 n-gram range
                indexSettings.put("analysis", analysis);

                Map<String, Object> settingsMap = new LinkedHashMap<>();
                settingsMap.put("index", indexSettings);

                Settings settings = Settings.parse(settingsMap);

                // ============================================================
                // Create Index and Apply Mappings
                // ============================================================
                boolean created = indexOps.create(settings);

                if (created) {
                    // Apply Disease.class field mappings (via @Field annotations)
                    indexOps.putMapping(indexOps.createMapping(Disease.class));
                    log.info("✓ Index {} created with analyzers, filters, and mappings",
                            DISEASES_INDEX_V1);
                } else {
                    log.warn("Index {} creation returned false (may already exist)",
                            DISEASES_INDEX_V1);
                }

            } catch (Exception ex) {
                // Graceful degradation: don't block app startup
                log.warn("⚠ Elasticsearch initialization failed: {}. "
                        + "App will boot with SQL fallback enabled. "
                        + "Error: {}", DISEASES_INDEX_V1, ex.getMessage());
                // Stack trace at debug level for troubleshooting
                log.debug("Elasticsearch initialization error details:", ex);
            }
        };
    }
}
```

**Key Points:**

| Component                       | Purpose               | Details                                               |
| ------------------------------- | --------------------- | ----------------------------------------------------- |
| `edge_ngram` filter             | Autocomplete matching | min_gram=1, max_gram=20 generates partial tokens      |
| `phonetic` filter               | Typo tolerance        | Double Metaphone encoding for sound-alike matches     |
| `disease_autocomplete` analyzer | Index-time analysis   | Applies lowercase + edge_ngram for searchable tokens  |
| `autocomplete_search` analyzer  | Query-time analysis   | Applies lowercase to normalize user input             |
| `phonetic_analyzer` analyzer    | Fallback search       | Applies phonetic encoding as last resort              |
| `max_ngram_diff=19`             | Safety limit          | Allows edge_ngram(1,20) range without rejection       |
| `CommandLineRunner` bean        | Auto-initialization   | Runs at Spring Boot startup, creates index if missing |
| `graceful degradation`          | Resilience            | Logs warning on ES failure, app continues with SQL    |

---

## 2.2: DiseaseSearchService Interface & Implementation

**Files to create:**

- `src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/DiseaseSearchService.java` (interface)
- `src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseSearchServiceImpl.java` (implementation)

### Interface Definition

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.service;

import com.artemhealthtech.bmc.medicodb.api.v1.user.dto.DiseaseDto;
import com.artemhealthtech.bmc.medicodb.api.v1.user.dto.PaginationRequestDTO;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

/**
 * Service interface for disease search operations via Elasticsearch.
 *
 * Provides high-level search methods with fuzzy matching, typo tolerance,
 * and weighted scoring. All implementation uses native Elasticsearch queries
 * via ElasticsearchOperations and BoolQueryBuilder for fine-grained control.
 */
public interface DiseaseSearchService {

    /**
     * Search for active diseases with fuzzy matching and autocomplete support.
     *
     * Uses weighted BoolQueryBuilder with multi-level ranking:
     * - Exact code match (12x boost)
     * - Prefix match (8x boost)
     * - Phrase match (6x boost)
     * - Autocomplete match (4x boost)
     * - Fuzzy match (2x boost)
     * - Phonetic match (1x boost)
     *
     * Deduplicates results per page on (diseaseName + "|" + diseaseCode).
     *
     * @param request PaginationRequestDTO with searchKey, page, size
     * @param pageable Spring Data Pageable for pagination
     * @return Page of DiseaseDto results, sorted by Elasticsearch score descending
     */
    Page<DiseaseDto> searchWithFuzzyAndAutocomplete(
            PaginationRequestDTO request,
            Pageable pageable);

    /**
     * Retrieve all active diseases without search filtering.
     *
     * Used for browse/admin workflows where no search key is provided.
     * Returns results in insertion order (Elasticsearch relevance not applicable).
     *
     * @param request PaginationRequestDTO with isActive flag
     * @param pageable Spring Data Pageable for pagination
     * @return Page of DiseaseDto results
     */
    Page<DiseaseDto> findAllActive(
            PaginationRequestDTO request,
            Pageable pageable);
}
```

### Implementation

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.service.impl;

import com.artemhealthtech.bmc.medicodb.api.v1.user.dto.DiseaseDto;
import com.artemhealthtech.bmc.medicodb.api.v1.user.dto.PaginationRequestDTO;
import com.artemhealthtech.bmc.medicodb.api.v1.user.service.DiseaseSearchService;
import com.hospisoft.model.Disease;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageImpl;
import org.springframework.data.domain.Pageable;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.SearchHit;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.data.elasticsearch.core.query.NativeSearchQuery;
import org.springframework.data.elasticsearch.core.query.NativeSearchQueryBuilder;
import org.springframework.stereotype.Service;
import co.elastic.clients.elasticsearch._types.query_dsl.BoolQuery;
import co.elastic.clients.elasticsearch._types.query_dsl.QueryBuilders;

import java.util.*;
import java.util.stream.Collectors;

/**
 * Service implementation for disease search via Elasticsearch.
 *
 * Uses BoolQueryBuilder with weighted scoring to rank results by relevance.
 * Implements multi-level query strategy:
 *
 * Query levels (in order of specificity):
 * 1. Exact code match (12x boost) - diseaseCode == value
 * 2. Prefix match (8x boost) - diseaseName starts with value
 * 3. Phrase match (6x boost) - diseaseName contains phrase as-is
 * 4. Autocomplete match (4x boost) - uses edge_ngram analyzer (keystroke match)
 * 5. Fuzzy match (2x boost) - 1-2 edits allowed (typo tolerance)
 * 6. Phonetic match (1x boost) - phonetic encoding (accent/phoneme tolerance)
 *
 * Deduplication:
 * HashSet<String> on (diseaseName + "|" + diseaseCode).toLowerCase() per page
 * keeps first occurrence, removes duplicates from Elasticsearch results.
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class DiseaseSearchServiceImpl implements DiseaseSearchService {

    private final ElasticsearchOperations elasticsearchOperations;

    @Override
    public Page<DiseaseDto> searchWithFuzzyAndAutocomplete(
            PaginationRequestDTO request,
            Pageable pageable) {

        log.debug("Elasticsearch search: searchKey='{}', page={}, size={}",
                request.getSearchKey(), pageable.getPageNumber(), pageable.getPageSize());

        String searchKey = request.getSearchKey().trim().toLowerCase();

        // ============================================================
        // Build Weighted BoolQuery with Multiple Scoring Levels
        // ============================================================
        BoolQuery.Builder boolBuilder = new BoolQuery.Builder();

        // Level 1: Exact code match (12x boost)
        // Matches disease code exactly or close to it
        boolBuilder.should(QueryBuilders.match()
                .field("diseaseCode")
                .query(searchKey)
                .fuzziness("0")
                .boost(12.0f)
                .build()
                ._toQuery());

        // Level 2: Prefix match (8x boost)
        // Disease name starts with search key
        boolBuilder.should(QueryBuilders.matchPhrase()
                .field("diseaseName")
                .query(searchKey)
                .boost(8.0f)
                .build()
                ._toQuery());

        // Level 3: Phrase match (6x boost)
        // All search terms must appear in disease name in order
        boolBuilder.should(QueryBuilders.match()
                .field("diseaseName")
                .query(searchKey)
                .operator(co.elastic.clients.elasticsearch._types.query_dsl.Operator.And)
                .boost(6.0f)
                .build()
                ._toQuery());

        // Level 4: Autocomplete match with edge_ngram (4x boost)
        // Keystroke-by-keystroke matching via edge_ngram analyzer
        // "d" → matches "diabetes", "dengue", etc.
        boolBuilder.should(QueryBuilders.match()
                .field("diseaseName")
                .query(searchKey)
                .analyzer("disease_autocomplete")
                .boost(4.0f)
                .build()
                ._toQuery());

        // Level 5: Fuzzy match for typos (2x boost)
        // Allows 1 edit distance (insert, delete, substitute)
        // "diabetus" → matches "diabetes"
        boolBuilder.should(QueryBuilders.match()
                .field("diseaseName")
                .query(searchKey)
                .fuzziness("1")
                .boost(2.0f)
                .build()
                ._toQuery());

        // Level 6: Phonetic fallback (1x boost)
        // Phonetic encoding (Double Metaphone) for accent/sound-alike
        // "colour" → matches "color" (phonetically similar)
        boolBuilder.should(QueryBuilders.match()
                .field("diseaseName")
                .query(searchKey)
                .analyzer("phonetic_analyzer")
                .boost(1.0f)
                .build()
                ._toQuery());

        // ============================================================
        // Filter: Only Active Diseases
        // ============================================================
        // All results must have isActive=true (hard filter, not scored)
        boolBuilder.filter(QueryBuilders.term()
                .field("isActive")
                .value(true)
                .build()
                ._toQuery());

        // ============================================================
        // Build and Execute Search Query
        // ============================================================
        NativeSearchQuery searchQuery = new NativeSearchQueryBuilder()
                .withQuery(boolBuilder.build()._toQuery())
                .withPageable(pageable)
                .build();

        // Execute search against Elasticsearch
        SearchHits<Disease> searchHits = elasticsearchOperations.search(
                searchQuery, Disease.class);

        // ============================================================
        // Convert to DTOs and Deduplicate
        // ============================================================
        List<DiseaseDto> results = convertAndDeduplicate(searchHits);

        log.debug("Search returned {} results (deduped from {})",
                results.size(), searchHits.getTotalHits());

        // Return as Spring Page with original pageable and total hits
        return new PageImpl<>(results, pageable, searchHits.getTotalHits());
    }

    @Override
    public Page<DiseaseDto> findAllActive(
            PaginationRequestDTO request,
            Pageable pageable) {

        log.debug("Elasticsearch browse all: page={}, size={}",
                pageable.getPageNumber(), pageable.getPageSize());

        // ============================================================
        // Simple match_all query filtered by isActive
        // ============================================================
        // No scoring/ranking (all docs equally relevant)
        // Used for admin/browse workflows without search terms
        NativeSearchQuery searchQuery = new NativeSearchQueryBuilder()
                .withQuery(QueryBuilders.bool()
                        .filter(QueryBuilders.term()
                                .field("isActive")
                                .value(true)
                                .build()
                                ._toQuery())
                        .build()
                        ._toQuery())
                .withPageable(pageable)
                .build();

        // Execute search
        SearchHits<Disease> searchHits = elasticsearchOperations.search(
                searchQuery, Disease.class);

        // Convert to DTOs (no deduplication needed for browse-all)
        List<DiseaseDto> results = searchHits.stream()
                .map(hit -> convertToDto(hit.getContent()))
                .collect(Collectors.toList());

        return new PageImpl<>(results, pageable, searchHits.getTotalHits());
    }

    /**
     * Convert Disease entity to DiseaseDto and deduplicate.
     * Keeps first occurrence of (diseaseName + "|" + diseaseCode), removes duplicates per page.
     *
     * Why deduplication: Some disease records may have multiple entries in DB
     * (historical reasons, data quality issues). ES returns all, we keep first.
     */
    private List<DiseaseDto> convertAndDeduplicate(SearchHits<Disease> searchHits) {
        Set<String> seen = new HashSet<>();
        List<DiseaseDto> results = new ArrayList<>();

        for (SearchHit<Disease> hit : searchHits) {
            Disease disease = hit.getContent();

            // Create deduplication key: combine name + code
            String dedupeKey = (disease.getDiseaseName() + "|" + disease.getDiseaseCode())
                    .toLowerCase();

            // Keep first occurrence, skip subsequent duplicates
            if (!seen.contains(dedupeKey)) {
                seen.add(dedupeKey);
                results.add(convertToDto(disease));
            }
        }

        return results;
    }

    /**
     * Convert Disease JPA entity to DiseaseDto for API response.
     */
    private DiseaseDto convertToDto(Disease disease) {
        return DiseaseDto.builder()
                .id(disease.getId())
                .diseaseName(disease.getDiseaseName())
                .diseaseCode(disease.getDiseaseCode())
                .diseaseGroup(disease.getDiseaseGroup())
                .isActive(disease.isActive())
                .isLinkedToHS(disease.isLinkedToHS())
                .isSeverityVisible(disease.getIsSeverityVisible())
                .snomedCTCode(disease.getSnomedCTCode())
                .icd11Code(disease.getIcd11Code())
                .build();
    }
}
```

### How It Works: Query Flow

| Level            | Boost | Condition                            | Example                          |
| ---------------- | ----- | ------------------------------------ | -------------------------------- |
| **Exact Code**   | 12x   | `diseaseCode == "ICD-A01"`           | User types code exactly          |
| **Prefix**       | 8x    | `diseaseName` starts with search key | "dia" matches "Diabetes"         |
| **Phrase**       | 6x    | ALL terms appear in order            | "viral fever" as phrase          |
| **Autocomplete** | 4x    | Edge n-gram tokens match             | "d", "di", "dia" keystroke match |
| **Fuzzy**        | 2x    | 1 edit distance allowed              | "diabetus" → "diabetes"          |
| **Phonetic**     | 1x    | Phonetic encoding match              | "colour" → "color"               |

**Scoring Example:**

```
Search: "diabetes"

Result 1: "Diabetes Mellitus Type 1"
  - Exact phrase match (6x) + Prefix match (8x) = combined high score ✓ Ranks #1

Result 2: "Diabetic Neuropathy"
  - Prefix match (8x) = medium score ✓ Ranks #2

Result 3: "Diabetus" (typo in DB)
  - Fuzzy match (2x) = low score ✓ Ranks #3

Results are deduplicated before returning to client.
```

---

## 3. Current Implementation (Before Changes)

### 3.1 Controller endpoint

The disease APIs are exposed from [DiseaseController.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/controller/DiseaseController.java#L78) and [DiseaseController.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/controller/DiseaseController.java#L119). The controller currently exposes fuzzy search endpoints and inherits generic pagination from `BaseController`.

Current controller code:

```java
// OLD CODE
@PostMapping("/paginate/fuzzy")
public ResponseEntity<ResponseDto> paginateDiseaseFuzzy(
        @RequestBody PaginationRequestDTO paginationRequestDTO) {
    Page<DiseaseDto> pageResult = service.paginateFuzzy(paginationRequestDTO);
    List<DiseaseDto> diseaseList = pageResult.getContent();
    return ResponseEntity.ok(CollectionResponseDto.builder()
            .responseCode(HttpStatus.OK.value())
            .responseObject(diseaseList)
            .totalRecords(pageResult.getTotalElements())
            .build());
}

// OLD CODE
@PostMapping("/fuzzy")
public ResponseEntity<ResponseDto> getDiseaseFuzzy(
        @RequestBody PaginationRequestDTO paginationRequestDTO) {
    List<DiseaseDto> diseaseList = service.getDiseaseFuzzy(paginationRequestDTO);
    return ResponseEntity.ok(CollectionResponseDto.builder()
            .responseCode(HttpStatus.OK.value())
            .responseObject(diseaseList)
            .totalRecords(diseaseList.isEmpty() ? 0L : (long) diseaseList.size())
            .build());
}
```

Line-level reference:

- `@PostMapping("/paginate/fuzzy")` at [DiseaseController.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/controller/DiseaseController.java#L78)
- `@PostMapping("/fuzzy")` at [DiseaseController.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/controller/DiseaseController.java#L119)
- the generic `POST /page` endpoint is inherited from [BaseController.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/controller/BaseController.java#L128)

### 3.2 Service method calling DB

The current main search path is in [DiseaseServiceImpl.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseServiceImpl.java#L171).

Current search flow:

```java
// OLD CODE
@Override
public Page<DiseaseDto> paginate(PaginationRequestDTO paginationRequestDTO) {
    Pageable pageable = PageRequest.of(
            paginationRequestDTO.getPage(),
            paginationRequestDTO.getSize()
    );

    if (ObjectUtils.isEmpty(paginationRequestDTO.getSearchKey())) {
        diseases = diseaseRepository.findByIsActive(paginationRequestDTO.getIsActive(), pageable);
    } else {
        String searchKey = paginationRequestDTO.getSearchKey().trim().toLowerCase();
        boolean isCodeSearch = searchKey.matches("^[a-z]\\d.*") || searchKey.contains(".");

        String diseaseCodeSearch = searchKey.replace(".", "");
        String containsPattern = "%" + searchKey + "%";
        String startsWithPattern = searchKey + "%";
        String codeContainsPattern = isCodeSearch ? "%" + diseaseCodeSearch + "%" : null;
        String codeStartsWithPattern = isCodeSearch ? diseaseCodeSearch + "%" : null;

        diseases = diseaseRepository.getDiseaseLikeAndIsActive(
                containsPattern,
                startsWithPattern,
                codeContainsPattern,
                codeStartsWithPattern,
                searchKey,
                isActive,
                pageable
        );
    }

    return new PageImpl<>(uniqueList, diseases.getPageable(), diseases.getTotalElements());
}
```

Line-level reference:

- `paginate(...)` at [DiseaseServiceImpl.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseServiceImpl.java#L171)

### 3.3 Existing fuzzy service methods

The current fuzzy behavior is still DB-backed and in-memory:

- [DiseaseServiceImpl.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseServiceImpl.java#L276) contains `paginateFuzzy(...)`
- [DiseaseServiceImpl.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseServiceImpl.java#L370) contains `getDiseaseFuzzy(...)`

These methods currently:

- For seen purpose and logic purpose
- These endpoints are not used at anywhere in frontend

### 3.4 Repository query using LIKE

The repository contains the SQL search layer in [DiseaseRepository.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/repository/DiseaseRepository.java#L20).

Relevant current code:

```java
// OLD CODE
@Query("SELECT d FROM Disease d " +
        "LEFT JOIN d.diseaseCategory dc " +
        "LEFT JOIN d.diseaseSubCategory dsc " +
        "LEFT JOIN d.healthStandard hs " +
        "WHERE (:isActive IS NULL OR d.isActive = :isActive) AND" +
        "(LOWER(d.diseaseName) LIKE :searchString " +
        "OR LOWER(d.diseaseCode) LIKE :searchString " +
        "OR LOWER(dc.diseaseCategory) LIKE :searchString " +
        "OR LOWER(dsc.diseaseSubCategory) LIKE :searchString " +
        "OR LOWER(hs.standardName) LIKE :searchString) ")
Page<Disease> findBySearchString(@Param("searchString") String searchString,
                                 @Param("isActive") Boolean isActive,
                                 Pageable pageable);
```

The actual main search method used by `paginate(...)` is the more complex native query at [DiseaseRepository.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/repository/DiseaseRepository.java#L125) and [DiseaseRepository.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/repository/DiseaseRepository.java#L199).

### 3.5 Current entity

The Disease entity is in the shared model module at [Disease.java](../../bmc-medico-models/src/main/java/com/hospisoft/model/Disease.java#L15).

Current model shape:

```java
// OLD CODE
@Entity
@Table(name = "tbdisease")
public class Disease extends CreateableEntity implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name="diseaseidp")
    private Long id;

    @Column(name="diseasename")
    private String diseaseName;

    @Column(name="diseasecode")
    private String diseaseCode;

    @ManyToOne(fetch = FetchType.LAZY)
    private DiseaseCategory diseaseCategory;

    @ManyToOne(fetch = FetchType.LAZY)
    private DiseaseSubCategory diseaseSubCategory;

    @ManyToOne(fetch = FetchType.LAZY)
    private HealthStandard healthStandard;
}
```

Line-level reference:

- JPA entity at [Disease.java](../../bmc-medico-models/src/main/java/com/hospisoft/model/Disease.java#L15)
- table mapping at [Disease.java](../../bmc-medico-models/src/main/java/com/hospisoft/model/Disease.java#L20)

---

## 4. Disadvantages of Current SQL Approach & What Elasticsearch Solves

### 4.1 Problems in the Current SQL Query

The current disease search uses complex SQL queries with `LIKE`, `REPLACE()`, and `soundex()` functions. These queries have fundamental limitations:

**1. Performance at Scale**

- `LIKE '%term%'` and multiple `OR` conditions force the database to scan many rows instead of using efficient text indices.
- As your disease catalog grows from thousands to millions, query latency increases exponentially.
- Example: `LOWER(disease_name) LIKE '%fracture%' OR LOWER(disease_code) LIKE '%fracture%'` requires a full table scan.

**2. CPU Overhead**

- The database must execute per-row functions on every query:
  - `LOWER()` — convert to lowercase
  - `REPLACE(disease_code, '.', '')` — normalize dot-delimited codes
  - `soundex(LOWER(disease_name))` — phonetic conversion
- With high query throughput (autocomplete traffic), this overhead becomes significant.

**3. Pagination Limitation**

- The query is paginated, so if a matching disease exists on page 5 but the user is viewing page 1, they won't see it unless they scroll through pages 1-4.
- Example: Search for "fracture" returns results sorted by relevance, but page 1 shows only the top 20 results; a better match on page 2 is invisible.
- This breaks user expectations for autocomplete and full-catalog search.

**4. No Autocomplete Support**

- The query doesn't support keystroke-driven prefix matching with progressive relevance scoring.
- Without an autocomplete index, typing `f-r-a` returns the same results as typing `fracture`; no incremental ranking.

**5. Hard to Tune**

- Adding new search fields or tweaking ranking logic requires modifying complex SQL with many `CASE` statements.
- Changes carry database expertise requirements and risk of breaking existing queries.
- Different search scenarios (exact code → prefix match → name contains) require different `ORDER BY` logic.

**6. No Synonym or Phonetic Support**

- The `soundex()` fallback is rigid and binary; either it matches or it doesn't.
- You cannot easily add synonym mappings like `PCM → Paracetamol` or `Flu → Influenza`.
- Phonetic variants require manual enum or lookup table maintenance.

**7. Result Ordering Inconsistency**

- Ranking logic is embedded in SQL with complex `CASE` statements.
- Different queries with different criteria need different `ORDER BY` logic.
- No unified scoring model across all search paths.

---

### 4.2 What Elasticsearch Solves

Elasticsearch is a purpose-built search engine with an inverted index architecture. Here's how it addresses each problem:

**1. Inverted Index — Orders of Magnitude Faster**

- Instead of scanning rows, Elasticsearch uses token postings. A search for `"fracture"` immediately knows which documents contain that word.
- No per-row function calls; the index is pre-computed at insert time.
- Query execution is `O(k log n)` instead of `O(n)` where `n` is the disease catalog size.

**2. Lower CPU — Pre-Computed Index**

- Search execution doesn't require `LOWER()`, `REPLACE()`, or `soundex()` on every row.
- The index is computed once at insert/update time, not on every query.
- CPU spent on the database drops significantly for high-traffic search scenarios.

**3. Full-Text Relevance — TF-IDF and BM25 Scoring**

- Elasticsearch ranks results by relevance using industry-standard algorithms (TF-IDF or BM25).
- The best matches naturally appear first without complex `CASE` statements.
- Example: "fracture of femur" is ranked higher than "femoral fracture" because "fracture" appears first and phrase proximity is closer.

**4. Autocomplete Built-In — Edge N-Gram Support**

- Using `edge_ngram` or `search_as_you_type` analyzers, the same index handles keystroke-driven autocomplete.
- Typing `f-r-a-c` returns progressively filtered results ranked by relevance.
- No need for separate autocomplete indices or complex pagination logic.

**5. Fuzzy Matching — `fuzziness: AUTO`**

- `fuzziness: AUTO` handles typos at query time, not after scanning the database.
- Levenshtein distance is computed during search, not after loading all records.
- Example: "frcture" (missing 'a') is automatically matched to "fracture".

**6. Analyzers and Synonyms — Configurable and Extensible**

- You can configure the index to understand synonyms, phonetic variants, and custom tokenization.
- Changes don't require SQL or database expertise.
- Example: Define a synonym filter that maps `PCM → Paracetamol` and `Paracet → Paracetamol`.
- Phonetic analyzer (Double Metaphone via `analysis-phonetic` plugin) understands similar sounding names.

**7. Consistent Ranking — Same Scoring Model for All Queries**

- All queries use the same relevance scoring model.
- You tune boosting factors in the query DSL, not in SQL `ORDER BY` clauses.
- Example: Boost exact code matches 12x, prefix matches 8x, phrase matches 6x, autocomplete 4x — all in one place.

**8. Better Pagination — Search Full Index, Not Just First Page**

- Elasticsearch searches across the entire index, not just the current database page.
- Page 5 results are just as fast to fetch as page 1 because the search engine knows all matches before pagination.

---

### 5.1 No first-class fuzzy handling

The code has fuzzy-like behavior, but it is implemented in Java and depends on loading data first.

Where it happens:

- [DiseaseServiceImpl.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseServiceImpl.java#L276)
- [DiseaseServiceImpl.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseServiceImpl.java#L370)

Impact:

- fuzzy matching is not handled by the search engine
- relevance tuning is difficult
- matching gets more expensive as the disease list grows

### 5.2 Tight coupling with DB

Search, filtering, ranking, and data retrieval are all bundled into database access. This makes the code harder to scale, tune, and isolate.

Where it happens:

- [DiseaseRepository.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/repository/DiseaseRepository.java#L20)
- [DiseaseServiceImpl.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseServiceImpl.java#L171)

Impact:

- search performance is bound to DB capacity
- query tuning becomes harder
- no proper autocomplete index
- no versioned search index or alias-based rollback

---

## 6. Elasticsearch Integration - Step-by-Step Code Changes

### 6.1 Step 1: Add Dependency

File: [pom.xml](../pom.xml)

Add the Elasticsearch starter inside the existing `<dependencies>` block, near the current Spring Data dependencies.

Current anchor point:

- `spring-boot-starter-data-jpa` at [pom.xml](../pom.xml#L120)

```xml
<!-- OLD CODE -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- NEW CODE -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

Explanation:

- This enables Spring Data Elasticsearch support.
- Keep the dependency version managed by the Spring Boot parent BOM.
- Do not remove JPA; DB remains the source of truth.

---

### 6.2 Step 2: Elasticsearch Configuration

File to add: [src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/config/ElasticsearchConfig.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/config/ElasticsearchConfig.java)

Recommended responsibilities:

- declare the Elasticsearch client connection
- enable Elasticsearch repositories
- read `spring.elasticsearch.uris`, `spring.elasticsearch.username`, and `spring.elasticsearch.password`
- wire the template/operations bean if you need custom queries

Example:

```java
// OLD CODE
// No Elasticsearch config exists today.

// NEW CODE
@Configuration
@EnableElasticsearchRepositories(basePackages = "com.artemhealthtech.bmc.medicodb.api.v1.user.repository")
public class ElasticsearchConfig {

    @Value("${spring.elasticsearch.uris}")
    private String elasticsearchUris;

    @Bean
    public ClientConfiguration clientConfiguration() {
        return ClientConfiguration.builder()
                .connectedTo(elasticsearchUris.replace("http://", "").replace("https://", ""))
                .build();
    }
}
```

Explanation:

- Keep the config minimal and environment-driven.
- If authentication is required, add basic auth in the builder.
- Prefer versioned index names and aliases in production.

---

### 6.3 Step 3: Entity Update

File: [Disease.java](../../bmc-medico-models/src/main/java/com/hospisoft/model/Disease.java)

Add Elasticsearch annotations and map search-friendly fields.

```java
// OLD CODE
@Entity
@Table(name = "tbdisease")
public class Disease extends CreateableEntity implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name="diseaseidp")
    private Long id;

    @Column(name="diseasename")
    private String diseaseName;

    @Column(name="diseasecode")
    private String diseaseCode;
}

// NEW CODE
@Document(indexName = "diseases_v1")
@Entity
@Table(name = "tbdisease")
public class Disease extends CreateableEntity implements Serializable {

    @Id
    @Field(type = FieldType.Long)
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name="diseaseidp")
    private Long id;

    @Field(type = FieldType.Text, analyzer = "disease_autocomplete", searchAnalyzer = "standard")
    @Column(name="diseasename")
    private String diseaseName;

    @Field(type = FieldType.Text, analyzer = "disease_autocomplete", searchAnalyzer = "standard")
    @Column(name="diseasecode")
    private String diseaseCode;

    @Field(type = FieldType.Keyword)
    private String diseaseCodeNormalized;

    @Field(type = FieldType.Boolean)
    @Column(name="isactive")
    private boolean isActive;
}
```

Explanation:

- `@Document` marks the class as indexable in Elasticsearch.
- `diseaseCodeNormalized` should store a dot-free code such as `S82899A`.
- The category and standard relationships should be flattened into ES-readable fields if they are needed for search or display.
- If you prefer to keep persistence concerns separated, the cleaner production approach is to create a dedicated document class, but this guide follows the requested entity update path.

---

### 6.4 Step 4: Create Elasticsearch Repository

File to add: [DiseaseSearchRepository.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/repository/DiseaseSearchRepository.java)

```java
// OLD CODE
// Search only exists in DiseaseRepository through JPA and native SQL.

// NEW CODE
@Repository
public interface DiseaseSearchRepository extends ElasticsearchRepository<Disease, Long> {
}
```

Explanation:

- Use ElasticsearchRepository for read/search access.
- Keep `DiseaseRepository` for DB writes and fallback support.
- Add a custom repository only if you want repository-level search methods beyond `ElasticsearchOperations`.

---

### 6.5 Step 5: Service Layer Refactor

File: [DiseaseServiceImpl.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseServiceImpl.java)

The service should stop building search logic around SQL `LIKE` queries and instead delegate search to Elasticsearch.

Current code to replace is at [DiseaseServiceImpl.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseServiceImpl.java#L171).

```java
// OLD CODE
if (ObjectUtils.isEmpty(paginationRequestDTO.getSearchKey())) {
    diseases = diseaseRepository.findByIsActive(paginationRequestDTO.getIsActive(), pageable);
} else {
    diseases = diseaseRepository.getDiseaseLikeAndIsActive(
            containsPattern,
            startsWithPattern,
            codeContainsPattern,
            codeStartsWithPattern,
            searchKey,
            isActive,
            pageable
    );
}

// NEW CODE
if (ObjectUtils.isEmpty(paginationRequestDTO.getSearchKey())) {
    return diseaseSearchService.findAllActive(paginationRequestDTO, pageable);
}

return diseaseSearchService.searchWithFuzzyAndAutocomplete(paginationRequestDTO, pageable);
```

Explanation:

- `DiseaseServiceImpl` should remain responsible for entity CRUD and DB persistence.
- `DiseaseSearchService` should own read/search behavior.
- Keep a DB fallback path if Elasticsearch is unavailable.
- Preserve the existing response DTO mapping so the API remains backward compatible.

Suggested service contract:

```java
public interface DiseaseSearchService {
    Page<DiseaseDto> findAllActive(PaginationRequestDTO request, Pageable pageable);
    Page<DiseaseDto> searchWithFuzzyAndAutocomplete(PaginationRequestDTO request, Pageable pageable);
}
```

---

### Step 6: Controller Changes

File: [DiseaseController.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/controller/DiseaseController.java)

The controller should remain a thin pass-through. The only behavioral change is that search should route into the Elasticsearch-backed service instead of SQL-based logic.

```java
// OLD CODE
Page<DiseaseDto> pageResult = service.paginateFuzzy(paginationRequestDTO);

// NEW CODE
Page<DiseaseDto> pageResult = diseaseSearchService.paginate(paginationRequestDTO);
```

For the non-paginated fuzzy endpoint:

```java
// OLD CODE
List<DiseaseDto> diseaseList = service.getDiseaseFuzzy(paginationRequestDTO);

// NEW CODE
List<DiseaseDto> diseaseList = diseaseSearchService.search(paginationRequestDTO);
```

Explanation:

- Keep the same endpoint shape and response wrapper.
- If you introduce a new dedicated search endpoint, use a non-breaking route such as `POST /api/v1/user/disease/search`.
- Avoid putting query-building logic in the controller.

---

### Step 7: Custom Query Implementation

File to add: [DiseaseSearchServiceImpl.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseSearchServiceImpl.java)

Use a custom Elasticsearch query so you can combine fuzzy search, prefix autocomplete, and boolean filters.

```java
// OLD CODE
// SQL LIKE + Java fuzzy matching.

// NEW CODE
BoolQueryBuilder boolQuery = QueryBuilders.boolQuery()
        .filter(QueryBuilders.termQuery("isActive", true))
        .should(QueryBuilders.termQuery("diseaseCode.keyword", searchKey).boost(12f))
        .should(QueryBuilders.matchPhrasePrefixQuery("diseaseName", searchKey).boost(6f))
        .should(QueryBuilders.matchQuery("diseaseName", searchKey)
                .fuzziness(Fuzziness.AUTO)
                .boost(3f))
        .should(QueryBuilders.matchQuery("diseaseCode", searchKey)
                .boost(4f))
        .minimumShouldMatch(1);

NativeSearchQuery query = new NativeSearchQueryBuilder()
        .withQuery(boolQuery)
        .withPageable(pageable)
        .build();
```

Recommended query behavior:

- exact code match should rank first
- prefix code and prefix name should support autocomplete
- fuzzy typo handling should be used as a lower-weight fallback
- `isActive = true` should always be a filter, not a scored clause

If you want broader ranking control, use a `should` chain like this:

1. exact `diseaseCode.keyword`
2. `match_phrase_prefix` on `diseaseName`
3. `match` with `fuzziness = AUTO`
4. `match` on normalized code

## 7. Before vs After Code Comparison

5. fallback to `match_all` only if you explicitly want broad discovery

---

### Pagination vs Fuzzy: problem & fixes

Problem

- Traditional DB-style pagination can hide relevant fuzzy matches that score lower in the raw DB result order and therefore fall outside page 1.
- Applying fuzzy matching after DB paging (or using per-row fuzzy checks) breaks global ranking and surprises users.

Fixes (practical, ordered)

1. Global ES ranking (recommended): run a single Elasticsearch query that combines exact, prefix, phrase, autocomplete and fuzzy clauses with weighted boosts so ES ranks the best fuzzy matches into page 1.
2. Separate autocomplete endpoint: call a small `autocomplete` API on keystroke (debounced) returning `size=8-12` suggestions; let `View all` hit the paginated `/page` endpoint.
3. Rescore top-N: execute a fast prefix/phrase query, then `rescore` the top 50 with a tolerant fuzzy query to bubble up good fuzzy matches without scanning the whole index.
4. Increase initial result window: return a larger first-page `size` (e.g., 30) and page client-side until exhausted, reducing the chance a fuzzy match is missed.
5. Avoid DB-style `from` deep paging; use `search_after` for stable deep pagination.

Concrete query examples

- Bool query with weighted clauses (drop into `DiseaseSearchServiceImpl`):

```java
BoolQueryBuilder b = QueryBuilders.boolQuery()
    .filter(QueryBuilders.termQuery("isActive", true))
    .should(QueryBuilders.termQuery("diseaseCode.keyword", searchKey).boost(12f))
    .should(QueryBuilders.matchPhrasePrefixQuery("diseaseName", searchKey).boost(6f))
    .should(QueryBuilders.matchQuery("diseaseName", searchKey).fuzziness(Fuzziness.AUTO).boost(2f))
    .should(QueryBuilders.matchQuery("diseaseName.autocomplete", searchKey).boost(4f))
    .minimumShouldMatch(1);

NativeSearchQuery q = new NativeSearchQueryBuilder()
    .withQuery(b)
    .withPageable(pageable)
    .build();
```

- Rescore example (rescore top-50 with fuzzy):

```java
RescoreBuilder rescore = RescoreBuilder.queryRescorer(
    QueryBuilders.matchQuery("diseaseName", searchKey).fuzziness(Fuzziness.AUTO)
).setWindowSize(50);

NativeSearchQuery q = new NativeSearchQueryBuilder()
    .withQuery(b)
    .withRescore(rescore)
    .withPageable(pageable)
    .build();
```

- Autocomplete endpoint (lightweight):

```
POST /api/v1/user/disease/autocomplete
Body: { "query": "typh", "size": 10 }
```

Frontend pattern (debounce + autocomplete + paginated "View all")

1. Debounce user input (250–350ms).
2. Call `/disease/autocomplete` for suggestions while typing; show top 8–12.
3. If user selects suggestion — navigate or fill field and fetch full record.
4. Offer a `View all results` action that opens the paginated search (`/disease/page`) using the same `searchKey` and server-side paging.

Notes

- Test `fuzziness`, `prefix_length`, and `max_expansions` to balance recall vs noise.
- Monitor autocomplete latency and set a 100–200ms SLA target for cached/common queries.

---

### Step 7.1: Method Placement and Empty Repository Clarification

The following methods:

```java
Page<DiseaseDto> findAllActive(PaginationRequestDTO request, Pageable pageable);
Page<DiseaseDto> searchWithFuzzyAndAutocomplete(PaginationRequestDTO request, Pageable pageable);
```

should be declared in:

- `src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/DiseaseSearchService.java`

and implemented in:

- `src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseSearchServiceImpl.java`

`DiseaseSearchRepository` may be intentionally minimal/empty at first:

- File: `src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/repository/DiseaseSearchRepository.java`
- It should simply extend `ElasticsearchRepository<Disease, Long>`.
- Spring Data already provides standard CRUD/index methods.
- Complex fuzzy/autocomplete/boosted queries are built in `DiseaseSearchServiceImpl` via Elasticsearch query builders, not as repository method names.

---

## 7. Before vs After Code Comparison

### SQL query vs Elasticsearch query

```java
// BEFORE: SQL/JPA
@Query(value = """
    SELECT d1_0.*
    FROM tbdisease d1_0
    WHERE (
        LOWER(d1_0.diseasename) LIKE :containsPattern
     OR LOWER(d1_0.diseasecode) LIKE :containsPattern
     OR LOWER(REPLACE(d1_0.diseasecode, '.', '')) LIKE :codeContainsPattern
     OR soundex(LOWER(d1_0.diseasename)) = soundex(LOWER(:rawSearchString))
    )
    AND d1_0.isactive = :isActive
    """, nativeQuery = true)
Page<Disease> getDiseaseLikeAndIsActive(...);

// AFTER: Elasticsearch
BoolQueryBuilder boolQuery = QueryBuilders.boolQuery()
        .filter(QueryBuilders.termQuery("isActive", true))
        .should(QueryBuilders.termQuery("diseaseCode.keyword", searchKey).boost(12f))
        .should(QueryBuilders.matchPhrasePrefixQuery("diseaseName", searchKey).boost(6f))
        .should(QueryBuilders.matchQuery("diseaseName", searchKey).fuzziness(Fuzziness.AUTO).boost(3f))
        .minimumShouldMatch(1);
```

### Old service vs new service

```java
// BEFORE: service reads DB and does in-memory fuzzy work
Page<Disease> diseases = diseaseRepository.getDiseaseLikeAndIsActive(...);
return new PageImpl<>(uniqueList, diseases.getPageable(), diseases.getTotalElements());

// AFTER: service delegates to search layer
Page<DiseaseDto> result = diseaseSearchService.searchWithFuzzyAndAutocomplete(request, pageable)
        .map(this::modelToDto);
```

---

## 8. Data Indexing Script

File to add: [DiseaseReindexService.java](../src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseReindexService.java)

This service pushes data from MySQL into Elasticsearch in batches.

```java
// OLD CODE
// No dedicated reindex job exists.

// NEW CODE
@Service
@RequiredArgsConstructor
public class DiseaseReindexService {

    private final DiseaseRepository diseaseRepository;
    private final DiseaseSearchRepository diseaseSearchRepository;

    @Transactional(readOnly = true)
    public void reindexAllDiseases() {
        int page = 0;
        int size = 500;
        Page<Disease> diseasePage;

        do {
            diseasePage = diseaseRepository.findAll(PageRequest.of(page, size, Sort.by("id")));
            List<Disease> batch = diseasePage.getContent();

            List<Disease> documents = batch.stream()
                    .peek(d -> d.setDiseaseCodeNormalized(
                            d.getDiseaseCode() == null ? null : d.getDiseaseCode().replace(".", "")
                    ))
                    .toList();

            diseaseSearchRepository.saveAll(documents);
            page++;
        } while (diseasePage.hasNext());
    }
}
```

Production notes:

- use batch sizes that match your index throughput
- for larger catalogs, prefer bulk operations
- run this from an admin endpoint, a job, or a controlled deployment task
- after a successful bulk load, switch an alias to the new index version

If you want incremental sync too, trigger `save` or `delete` in Elasticsearch after the DB transaction commits.

---

## 8.1 Incremental Sync: Add/Update/Delete Hooks

**This is critical for production.** Every time a disease is created, updated, or deleted in the database, the Elasticsearch index must be kept in sync.

### File to add: DiseaseIndexingService.java

Create a dedicated service to handle all Elasticsearch sync operations:

```java
// NEW CODE
@Service
@RequiredArgsConstructor
public class DiseaseIndexingService {

    private final DiseaseSearchRepository diseaseSearchRepository;
    private static final Logger logger = LoggerFactory.getLogger(DiseaseIndexingService.class);

    /**
     * Index a disease document in Elasticsearch after DB commit
     */
    public void indexDisease(Disease disease) {
        try {
            if (disease == null) return;

            // Normalize code (remove dots)
            if (disease.getDiseaseCode() != null) {
                disease.setDiseaseCodeNormalized(disease.getDiseaseCode().replace(".", ""));
            }

            // Only index active diseases
            if (disease.isActive()) {
                diseaseSearchRepository.save(disease);
                logger.info("Indexed disease: id={}, name={}", disease.getId(), disease.getDiseaseName());
            } else {
                // Remove inactive diseases from index
                deleteDiseaseFromIndex(disease.getId());
            }
        } catch (Exception e) {
            logger.error("Failed to index disease: id={}", disease.getId(), e);
            // Don't throw; ES sync should not block DB operations
        }
    }

    /**
     * Remove a disease document from Elasticsearch after DB delete/deactivate
     */
    public void deleteDiseaseFromIndex(Long diseaseId) {
        try {
            if (diseaseId == null) return;
            diseaseSearchRepository.deleteById(diseaseId);
            logger.info("Deleted disease from index: id={}", diseaseId);
        } catch (Exception e) {
            logger.error("Failed to delete disease from index: id={}", diseaseId, e);
            // Don't throw; ES sync should not block DB operations
        }
    }

    /**
     * Reindex multiple diseases (batch operation)
     */
    public void indexDiseases(List<Disease> diseases) {
        try {
            if (diseases == null || diseases.isEmpty()) return;

            List<Disease> activeDiseasesToIndex = diseases.stream()
                    .peek(d -> {
                        if (d.getDiseaseCode() != null) {
                            d.setDiseaseCodeNormalized(d.getDiseaseCode().replace(".", ""));
                        }
                    })
                    .filter(Disease::isActive)
                    .toList();

            diseaseSearchRepository.saveAll(activeDiseasesToIndex);
            logger.info("Indexed {} diseases", activeDiseasesToIndex.size());
        } catch (Exception e) {
            logger.error("Failed to batch index diseases", e);
        }
    }
}
```

---

### Modify: DiseaseServiceImpl.java - Add hooks to save() and delete()

Update the disease service to call `DiseaseIndexingService` after DB commit:

```java
// BEFORE: DiseaseServiceImpl.save()
@Override
public DiseaseDto save(DiseaseDto diseaseDto) {
    Disease disease = mapper.map(diseaseDto, Disease.class);
    Disease saved = diseaseRepository.save(disease);  // DB save only
    return mapper.map(saved, DiseaseDto.class);
}

// AFTER: DiseaseServiceImpl.save()
@Override
public DiseaseDto save(DiseaseDto diseaseDto) {
    Disease disease = mapper.map(diseaseDto, Disease.class);
    Disease saved = diseaseRepository.save(disease);

    // ✅ NEW: Index in Elasticsearch after DB commit
    diseaseIndexingService.indexDisease(saved);

    return mapper.map(saved, DiseaseDto.class);
}
```

```java
// BEFORE: DiseaseServiceImpl.update()
@Override
public DiseaseDto update(Long id, DiseaseDto diseaseDto) {
    Disease disease = diseaseRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Disease not found"));
    mapper.map(diseaseDto, disease);
    Disease updated = diseaseRepository.save(disease);  // DB update only
    return mapper.map(updated, DiseaseDto.class);
}

// AFTER: DiseaseServiceImpl.update()
@Override
public DiseaseDto update(Long id, DiseaseDto diseaseDto) {
    Disease disease = diseaseRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Disease not found"));
    mapper.map(diseaseDto, disease);
    Disease updated = diseaseRepository.save(disease);

    // ✅ NEW: Reindex in Elasticsearch after DB update
    diseaseIndexingService.indexDisease(updated);

    return mapper.map(updated, DiseaseDto.class);
}
```

```java
// BEFORE: DiseaseServiceImpl.delete()
@Override
public void delete(Long id) {
    diseaseRepository.deleteById(id);  // DB delete only
}

// AFTER: DiseaseServiceImpl.delete()
@Override
public void delete(Long id) {
    diseaseRepository.deleteById(id);

    // ✅ NEW: Remove from Elasticsearch index after DB delete
    diseaseIndexingService.deleteDiseaseFromIndex(id);
}
```

---

### Alternative: Use @TransactionalEventListener (Advanced)

If you prefer event-driven sync (cleaner separation of concerns):

```java
// NEW CODE: Create a listener in DiseaseServiceImpl or separate file
@Service
@RequiredArgsConstructor
public class DiseaseIndexingSyncListener {

    private final DiseaseIndexingService diseaseIndexingService;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onDiseaseCreated(DiseaseCreatedEvent event) {
        diseaseIndexingService.indexDisease(event.getDisease());
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onDiseaseUpdated(DiseaseUpdatedEvent event) {
        diseaseIndexingService.indexDisease(event.getDisease());
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onDiseaseDeleted(DiseaseDeletedEvent event) {
        diseaseIndexingService.deleteDiseaseFromIndex(event.getDiseaseId());
    }
}
```

Then modify service to publish events:

```java
@Service
public class DiseaseServiceImpl implements DiseaseService {

    private final ApplicationEventPublisher eventPublisher;

    @Override
    public DiseaseDto save(DiseaseDto diseaseDto) {
        Disease disease = mapper.map(diseaseDto, Disease.class);
        Disease saved = diseaseRepository.save(disease);

        // ✅ Publish event (listener will handle ES sync after commit)
        eventPublisher.publishEvent(new DiseaseCreatedEvent(saved));

        return mapper.map(saved, DiseaseDto.class);
    }
}
```

---

### Recommended Approach for Your Project

**Option 1 (Simple - Recommended for Phase 1):**

- Call `diseaseIndexingService.indexDisease()` directly after `save()`
- No extra event classes needed
- Easy to understand and debug

**Option 2 (Advanced - For Phase 2+):**

- Use `@TransactionalEventListener` for event-driven sync
- Better separation of concerns
- Easier to add additional listeners later (logging, auditing, etc.)

---

## 10. Testing the Implementation

### Example API request

```http
POST /api/v1/user/disease/paginate/fuzzy
Content-Type: application/json

{
  "page": 0,
  "size": 10,
  "searchKey": "fractr",
  "isActive": true
}
```

### Example response

```json
{
  "responseCode": 200,
  "responseObject": [
    {
      "id": 101,
      "diseaseName": "Fracture of shaft of femur",
      "diseaseCode": "S72.3",
      "diseaseCategory": "Orthopedic",
      "diseaseSubCategory": "Fracture"
    }
  ],
  "totalRecords": 1
}
```

Recommended verification checklist:

- exact disease code search
- code prefix search
- name autocomplete search
- typo-tolerant search
- inactive disease hidden from results
- DB fallback when Elasticsearch is unavailable

---

## 11. Performance Impact (Code Perspective)

The performance model changes significantly:

- SQL `LIKE '%term%'` usually causes a table or range scan on the database side.
- Elasticsearch uses an inverted index, so query execution starts from token postings rather than scanning rows.
- `match_phrase_prefix` is much better for autocomplete than re-checking every record in Java.
- `fuzziness = AUTO` gives typo tolerance at query time instead of in-memory comparison after fetching records.

Code-level outcome:

- less DB CPU for search traffic
- better latency consistency for keystroke-driven search
- more predictable relevance scoring
- easier tuning for autocomplete and fuzzy matches

---

## 12. Important Notes for Developers

### Naming conventions

- Use a versioned index name such as `diseases_v1`.
- Keep a stable alias such as `diseases` for runtime reads.
- Normalize code fields into a separate keyword field like `diseaseCodeNormalized`.

### Index versioning

- Build `diseases_v2` when mappings change.
- Bulk load the new index.
- Validate counts and sample searches.
- Move the alias atomically.
- Retire the old index after verification.

### Error handling

- If Elasticsearch is down, fall back to the current SQL repository.
- Do not fail the API if search can safely degrade.
- Separate search failures from DB write failures.

### Logging

- Log the search mode, not sensitive payloads.
- Include query length, index version, and elapsed time.
- Do not log PHI or unnecessary request details.

---

## 13. Rollback Strategy

Rollback should be simple and low risk.

Recommended rollback plan:

1. Disable the Elasticsearch feature flag.
2. Route the controller back to the DB-backed service path.
3. Keep the API response shape unchanged so clients do not need to change.
4. Leave the ES index in place until the rollback is verified.
5. Remove the Elasticsearch dependency only after the DB path is confirmed stable.

Practical fallback design:

```java
// OLD CODE
return diseaseSearchService.searchWithFuzzyAndAutocomplete(request, pageable);

// ROLLBACK CODE
return diseaseRepository.getDiseaseLikeAndIsActive(...);
```

This makes rollback a configuration or service-routing change instead of a full API rewrite.

---

## Summary

The current Disease Search is DB-bound and uses `LIKE`, native SQL, and Java-side fuzzy logic. The Elasticsearch migration should move the search path into a dedicated search service, use versioned indices with aliases, keep DB writes unchanged, and preserve the existing API contract.

For implementation, start with:

1. add the Elasticsearch dependency
2. add the Elasticsearch config
3. annotate or project the Disease entity into an indexable search model
4. add the Elasticsearch repository
5. move search logic into `DiseaseSearchService`
6. keep the controller response contract unchanged
7. add a batch reindex job and a fallback path
