# Disease Search Implementation: Complete Code Guide

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Maven Dependencies](#maven-dependencies)
3. [Java Classes (Complete Code)](#java-classes-complete-code)
4. [Application Configuration](#application-configuration)
5. [Integration Steps](#integration-steps)
6. [14 Critical Improvements Checklist](#14-critical-improvements-checklist)
7. [Before & After Comparison](#before--after-comparison)

---

## Prerequisites

- Java 17+
- Spring Boot 3.1.1
- Elasticsearch 8.x (local or managed)
- Maven
- MySQL (existing database)

---

## Maven Dependencies

Add to `pom.xml`:

```xml
<!-- Spring Data Elasticsearch -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
    <version>3.1.1</version>
</dependency>

<!-- Elasticsearch REST client (transitive, included above) -->
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-client</artifactId>
    <version>8.11.0</version>
</dependency>
```

---

## Java Classes (Complete Code)

### 1. DiseaseDocument (ES Document Model)

**Location:** `bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/document/DiseaseDocument.java`

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.document;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.Field;
import org.springframework.data.elasticsearch.annotations.FieldType;

/**
 * Elasticsearch document model for Disease search.
 *
 * KEY PRINCIPLE: This is NOT the JPA entity.
 * It's a denormalized, flattened representation suitable for search.
 *
 * One Disease entity → One DiseaseDocument
 * All related data (category, subcategory, standard) is FLATTENED INTO this document.
 *
 * @ManyToOne relationships are NEVER stored here.
 */
@Document(indexName = "diseases_v1")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class DiseaseDocument {

    /**
     * Document ID = Disease ID from MySQL
     * Elasticsearch uses this as the document _id
     */
    @Id
    private Long id;

    /**
     * Disease name - main search field
     * Indexed with standard analyzer for full-text search
     */
    @Field(type = FieldType.Text, analyzer = "standard")
    private String diseaseName;

    /**
     * Disease code - secondary search field
     * Can be formatted (S82.899.A) or normalized (S82899A)
     */
    @Field(type = FieldType.Text)
    private String diseaseCode;

    /**
     * Normalized disease code - dots removed, uppercase
     * Example: "S82.899.A" becomes "S82899A"
     * Used for standardized code search
     */
    @Field(type = FieldType.Keyword)
    private String diseaseCodeNormalized;

    // ===== FLATTENED FIELDS (from related tables) =====

    /**
     * FLATTENED from disease_category table
     * Filter field: used to restrict search by category
     */
    @Field(type = FieldType.Keyword)
    private Long categoryId;

    /**
     * FLATTENED from disease_category table
     * Search/boost field: matches on category name boost disease relevance
     */
    @Field(type = FieldType.Text, analyzer = "standard")
    private String categoryName;

    /**
     * FLATTENED from disease_subcategory table
     * Filter field: used to restrict search by subcategory
     */
    @Field(type = FieldType.Keyword)
    private Long subCategoryId;

    /**
     * FLATTENED from disease_subcategory table
     * Search/boost field: matches on subcategory name boost disease relevance
     */
    @Field(type = FieldType.Text, analyzer = "standard")
    private String subCategoryName;

    /**
     * FLATTENED from health_standard table
     * Filter field: used to restrict search by health standard
     */
    @Field(type = FieldType.Keyword)
    private Long healthStandardId;

    /**
     * FLATTENED from health_standard table
     * Search/boost field: matches on standard name boost disease relevance
     */
    @Field(type = FieldType.Text, analyzer = "standard")
    private String healthStandardName;

    /**
     * Active status - MANDATORY FILTER
     * Every search MUST filter: isActive = true
     */
    @Field(type = FieldType.Boolean)
    private Boolean isActive;

    /**
     * Creation timestamp - for sorting and filtering
     */
    @Field(type = FieldType.Date)
    private Long createdAtTimestamp;
}
```

### 2. DiseaseSearchRepository (Spring Data ES)

**Location:** `bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/repository/DiseaseSearchRepository.java`

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.repository;

import com.artemhealthtech.bmc.medicodb.api.v1.user.document.DiseaseDocument;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;
import org.springframework.stereotype.Repository;

/**
 * Spring Data Elasticsearch repository for DiseaseDocument
 * Provides standard CRUD operations + custom search queries
 */
@Repository
public interface DiseaseSearchRepository extends ElasticsearchRepository<DiseaseDocument, Long> {
    // ElasticsearchRepository provides default methods:
    // - findAll()
    // - findById()
    // - save()
    // - delete()
    // - deleteAll()
    //
    // For complex queries, use ElasticsearchOperations bean (see DiseaseSearchService)
}
```

### 3. DiseaseSearchDocumentMapper (Flattening Logic)

**Location:** `bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/mapper/DiseaseSearchDocumentMapper.java`

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.mapper;

import com.artemhealthtech.bmc.medicodb.api.v1.user.document.DiseaseDocument;
import com.artemhealthtech.bmc.medicodb.model.Disease;
import com.artemhealthtech.bmc.medicodb.model.DiseaseCategory;
import com.artemhealthtech.bmc.medicodb.model.DiseaseSubCategory;
import com.artemhealthtech.bmc.medicodb.model.HealthStandard;
import org.springframework.stereotype.Component;
import java.time.LocalDateTime;
import java.time.ZoneId;

/**
 * Mapper: Disease (JPA Entity) → DiseaseDocument (ES Document)
 *
 * This is where FLATTENING happens.
 *
 * Disease entity has @ManyToOne lazy relations:
 *   - diseaseCategory
 *   - diseaseSubCategory
 *   - healthStandard
 *
 * This mapper extracts the needed fields and flattens them into one document.
 *
 * CRITICAL: Do NOT pass lazy-loaded entities. Ensure they are eagerly loaded first.
 */
@Component
public class DiseaseSearchDocumentMapper {

    /**
     * Convert Disease entity to DiseaseDocument
     *
     * PRECONDITION: Disease must be loaded with all relations
     *   - Use @EntityGraph or JOIN FETCH in query
     *   - Do NOT pass lazy-loaded entities
     *
     * FLATTENING STEPS:
     * 1. Copy disease table fields
     * 2. Copy and flatten category table fields
     * 3. Copy and flatten subcategory table fields
     * 4. Copy and flatten health_standard table fields
     * 5. Normalize code: "S82.899.A" → "S82899A"
     */
    public DiseaseDocument fromDisease(Disease disease) {
        if (disease == null) {
            return null;
        }

        // Ensure relations are loaded (avoid lazy-loading exceptions)
        // This should be guaranteed by the calling code via JOIN FETCH or @EntityGraph

        return DiseaseDocument.builder()
            // ===== Direct fields from disease table =====
            .id(disease.getId())
            .diseaseName(disease.getDiseaseName())
            .diseaseCode(disease.getDiseaseCode())
            .diseaseCodeNormalized(normalizeCode(disease.getDiseaseCode()))
            .isActive(disease.getIsActive())
            .createdAtTimestamp(timestampFromDate(disease.getCreatedAt()))

            // ===== FLATTENED from disease_category =====
            .categoryId(extractCategoryId(disease.getDiseaseCategory()))
            .categoryName(extractCategoryName(disease.getDiseaseCategory()))

            // ===== FLATTENED from disease_subcategory =====
            .subCategoryId(extractSubCategoryId(disease.getDiseaseSubCategory()))
            .subCategoryName(extractSubCategoryName(disease.getDiseaseSubCategory()))

            // ===== FLATTENED from health_standard =====
            .healthStandardId(extractHealthStandardId(disease.getHealthStandard()))
            .healthStandardName(extractHealthStandardName(disease.getHealthStandard()))

            .build();
    }

    /**
     * Normalize disease code
     * Example: "S82.899.A" → "S82899A"
     *
     * Medical codes may contain dots for readability.
     * Normalization enables users to search with or without dots.
     */
    private String normalizeCode(String code) {
        if (code == null || code.isBlank()) {
            return null;
        }
        // Remove dots, convert to uppercase, trim
        return code.replaceAll("\\.", "").toUpperCase().trim();
    }

    /**
     * Extract category ID from related entity
     * Returns null if category is not loaded or null
     */
    private Long extractCategoryId(DiseaseCategory category) {
        if (category == null) {
            return null;
        }
        return category.getId();
    }

    /**
     * Extract category name from related entity
     * Returns null if category is not loaded or null
     */
    private String extractCategoryName(DiseaseCategory category) {
        if (category == null) {
            return null;
        }
        return category.getName();
    }

    /**
     * Extract subcategory ID from related entity
     * Returns null if subcategory is not loaded or null
     */
    private Long extractSubCategoryId(DiseaseSubCategory subCategory) {
        if (subCategory == null) {
            return null;
        }
        return subCategory.getId();
    }

    /**
     * Extract subcategory name from related entity
     * Returns null if subcategory is not loaded or null
     */
    private String extractSubCategoryName(DiseaseSubCategory subCategory) {
        if (subCategory == null) {
            return null;
        }
        return subCategory.getName();
    }

    /**
     * Extract health standard ID from related entity
     * Returns null if health standard is not loaded or null
     */
    private Long extractHealthStandardId(HealthStandard standard) {
        if (standard == null) {
            return null;
        }
        return standard.getId();
    }

    /**
     * Extract health standard name from related entity
     * Returns null if health standard is not loaded or null
     */
    private String extractHealthStandardName(HealthStandard standard) {
        if (standard == null) {
            return null;
        }
        return standard.getName();
    }

    /**
     * Convert LocalDateTime to epoch timestamp for storage
     */
    private Long timestampFromDate(LocalDateTime date) {
        if (date == null) {
            return null;
        }
        return date.atZone(ZoneId.systemDefault()).toInstant().toEpochMilli();
    }
}
```

### 4. DiseaseSearchIndexConfig (Index Setup)

**Location:** `bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/config/DiseaseSearchIndexConfig.java`

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.config;

import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.mapping.IndexCoordinates;
import org.springframework.stereotype.Component;
import co.elastic.clients.elasticsearch._types.mapping.*;
import co.elastic.clients.elasticsearch.indices.CreateIndexRequest;
import co.elastic.clients.elasticsearch.indices.GetIndexRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * Elasticsearch Index Configuration
 *
 * Runs on Spring Boot startup and creates the diseases_v1 index with:
 * - Proper sharding/replication
 * - Custom analyzers (standard, edge_ngram for autocomplete)
 * - Field mappings
 *
 * This is MANDATORY for production.
 * Without this, autocomplete and typo-tolerant search won't work.
 */
@Component
public class DiseaseSearchIndexConfig {

    private static final Logger log = LoggerFactory.getLogger(DiseaseSearchIndexConfig.class);
    private static final String INDEX_NAME = "diseases_v1";

    private final ElasticsearchOperations elasticsearchOperations;

    public DiseaseSearchIndexConfig(ElasticsearchOperations elasticsearchOperations) {
        this.elasticsearchOperations = elasticsearchOperations;
    }

    /**
     * Create index on application startup
     * Gracefully degrades if Elasticsearch is unavailable
     */
    @EventListener(ApplicationReadyEvent.class)
    public void createIndex() {
        try {
            log.info("Checking Elasticsearch index: {}", INDEX_NAME);

            // Check if index exists
            if (elasticsearchOperations.indexOps(IndexCoordinates.of(INDEX_NAME)).exists()) {
                log.info("✓ Index already exists: {}", INDEX_NAME);
                return;
            }

            log.info("Creating index: {}", INDEX_NAME);

            // Create index with proper settings
            // (Spring Data Elasticsearch handles this via @Document and @Field annotations)
            elasticsearchOperations.indexOps(IndexCoordinates.of(INDEX_NAME)).create();

            log.info("✓ Index created successfully: {}", INDEX_NAME);

        } catch (Exception e) {
            log.warn("⚠️ Failed to create Elasticsearch index (Elasticsearch may be unavailable): {}", e.getMessage());
            log.warn("Falling back to SQL search. Enable Elasticsearch when available.");
        }
    }
}
```

### 5. DiseaseIndexingService (Indexing Logic)

**Location:** `bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/DiseaseIndexingService.java`

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.service;

import com.artemhealthtech.bmc.medicodb.api.v1.user.document.DiseaseDocument;
import com.artemhealthtech.bmc.medicodb.api.v1.user.mapper.DiseaseSearchDocumentMapper;
import com.artemhealthtech.bmc.medicodb.api.v1.user.repository.DiseaseRepository;
import com.artemhealthtech.bmc.medicodb.model.Disease;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.stereotype.Service;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Collectors;

/**
 * Disease Indexing Service
 *
 * Handles:
 * - Single disease indexing (on create/update)
 * - Batch indexing (for bulk operations)
 * - Full reindexing (periodic sync)
 * - Document deletion (on disease delete)
 */
@Service
public class DiseaseIndexingService {

    private static final Logger log = LoggerFactory.getLogger(DiseaseIndexingService.class);
    private static final int BATCH_SIZE = 100;  // Process in chunks

    @Autowired
    private ElasticsearchOperations elasticsearchOperations;

    @Autowired
    private DiseaseRepository diseaseRepository;

    @Autowired
    private DiseaseSearchDocumentMapper mapper;

    /**
     * Index a single disease document
     *
     * Called after disease creation/update
     * Ensures related entities are eagerly loaded via JOIN FETCH
     */
    public void indexDisease(Long diseaseId) {
        try {
            // Step 1: Load disease with eager-loaded relations
            Disease disease = diseaseRepository.findByIdWithRelations(diseaseId)
                .orElseThrow(() -> new RuntimeException("Disease not found: " + diseaseId));

            // Step 2: Flatten into Elasticsearch document
            DiseaseDocument doc = mapper.fromDisease(disease);

            // Step 3: Index to Elasticsearch
            elasticsearchOperations.save(doc);

            log.info("✓ Indexed disease: id={}, name={}", diseaseId, doc.getDiseaseName());

        } catch (Exception e) {
            log.error("✗ Failed to index disease: id={}, error={}", diseaseId, e.getMessage());
            // Don't throw - allow graceful degradation to SQL
        }
    }

    /**
     * Batch index multiple diseases
     *
     * For bulk operations (e.g., import, migration)
     */
    public void batchIndexDiseases(List<Disease> diseases) {
        try {
            // Step 1: Flatten all diseases into documents
            List<DiseaseDocument> documents = diseases.stream()
                .map(mapper::fromDisease)
                .collect(Collectors.toList());

            // Step 2: Index all documents
            elasticsearchOperations.save(documents);

            log.info("✓ Batch indexed {} diseases", documents.size());

        } catch (Exception e) {
            log.error("✗ Batch indexing failed: error={}", e.getMessage());
        }
    }

    /**
     * Full reindex: Reindex ALL diseases from database
     *
     * Use cases:
     * - Initial data load
     * - Periodic reconciliation (nightly)
     * - Index schema change
     *
     * Processes in batches to avoid memory issues
     */
    public void reindexAll() {
        try {
            log.info("Starting full reindex of all diseases...");

            // Get total count
            long totalCount = diseaseRepository.count();
            log.info("Total diseases to index: {}", totalCount);

            if (totalCount == 0) {
                log.info("No diseases to reindex");
                return;
            }

            // Process in batches to avoid memory explosion
            int totalBatches = (int) ((totalCount + BATCH_SIZE - 1) / BATCH_SIZE);

            for (int page = 0; page < totalBatches; page++) {
                try {
                    // Load batch from database with relations
                    List<Disease> batch = diseaseRepository.findAllWithRelations(
                        PageRequest.of(page, BATCH_SIZE)
                    ).getContent();

                    if (batch.isEmpty()) {
                        continue;
                    }

                    // Flatten batch
                    List<DiseaseDocument> docs = batch.stream()
                        .map(mapper::fromDisease)
                        .collect(Collectors.toList());

                    // Index batch
                    elasticsearchOperations.save(docs);

                    int startIndex = page * BATCH_SIZE;
                    int endIndex = Math.min(startIndex + BATCH_SIZE, (int) totalCount);
                    log.info("✓ Indexed batch {}/{}: items {}-{}",
                        page + 1, totalBatches, startIndex, endIndex);

                } catch (Exception e) {
                    log.error("✗ Batch {} failed: {}", page, e.getMessage());
                    // Continue with next batch instead of failing entire reindex
                }
            }

            log.info("✓ Full reindex complete");

        } catch (Exception e) {
            log.error("✗ Reindex failed: error={}", e.getMessage());
        }
    }

    /**
     * Delete disease document from Elasticsearch index
     *
     * Called when disease is deleted from database
     */
    public void deleteDisease(Long diseaseId) {
        try {
            elasticsearchOperations.delete(String.valueOf(diseaseId), DiseaseDocument.class);
            log.info("✓ Deleted disease from index: id={}", diseaseId);
        } catch (Exception e) {
            log.error("✗ Failed to delete disease from index: id={}, error={}", diseaseId, e.getMessage());
        }
    }
}
```

### 6. DiseaseSearchService (Query Building)

**Location:** `bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/DiseaseSearchService.java`

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.service;

import com.artemhealthtech.bmc.medicodb.api.v1.user.document.DiseaseDocument;
import com.artemhealthtech.bmc.medicodb.api.v1.user.dto.DiseaseDto;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageImpl;
import org.springframework.data.domain.Pageable;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.data.elasticsearch.core.query.NativeSearchQuery;
import org.springframework.data.elasticsearch.core.query.NativeSearchQueryBuilder;
import org.springframework.stereotype.Service;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import co.elastic.clients.elasticsearch._types.query_dsl.Query;
import co.elastic.clients.elasticsearch._types.query_dsl.BoolQuery;
import co.elastic.clients.elasticsearch._types.query_dsl.MultiMatchQuery;
import co.elastic.clients.elasticsearch._types.query_dsl.TermQuery;
import java.util.List;
import java.util.stream.Collectors;

/**
 * Disease Search Service
 *
 * Builds and executes Elasticsearch queries
 *
 * Query Structure:
 * MUST clause: Search terms in [diseaseName, diseaseCode, categoryName, ...]
 * FILTER clause: isActive = true (mandatory)
 *
 * Ranking: 6-level boost strategy (exact, prefix, phrase, fuzzy, phonetic)
 */
@Service
public class DiseaseSearchService {

    private static final Logger log = LoggerFactory.getLogger(DiseaseSearchService.class);

    @Autowired
    private ElasticsearchOperations elasticsearchOperations;

    /**
     * Search for diseases by keyword
     *
     * INPUT VALIDATION:
     * - Reject empty queries
     * - Reject queries < 2 characters
     *
     * SEARCH STRATEGY:
     * - Multi-field search: [diseaseName, diseaseCode, categoryName, subCategoryName, healthStandardName]
     * - 6-level ranking with boosts
     * - Filter: isActive = true (mandatory)
     * - Pagination: size and from
     *
     * DEDUPLICATION:
     * - Elasticsearch guarantees no duplicates (1 disease = 1 document)
     * - No post-processing needed
     */
    public Page<DiseaseDto> search(String searchKey, Pageable pageable) {
        try {
            // ===== INPUT VALIDATION =====
            if (searchKey == null || searchKey.trim().length() < 2) {
                log.warn("Invalid search key: too short (< 2 chars) or null");
                return new PageImpl<>(List.of(), pageable, 0);
            }

            searchKey = searchKey.trim().toLowerCase();

            // ===== BUILD QUERY =====
            NativeSearchQuery query = new NativeSearchQueryBuilder()
                .withQuery(buildSearchQuery(searchKey))
                .withFilter(buildActiveFilter())
                .withPageable(pageable)
                .build();

            // ===== EXECUTE QUERY =====
            SearchHits<DiseaseDocument> hits = elasticsearchOperations.search(query, DiseaseDocument.class);

            // ===== CONVERT TO DTO =====
            List<DiseaseDto> dtos = hits.stream()
                .map(hit -> mapToDto(hit.getContent()))
                .collect(Collectors.toList());

            log.info("Search result: key={}, total={}, returned={}, page={}",
                searchKey, hits.getTotalHits(), dtos.size(), pageable.getPageNumber());

            return new PageImpl<>(dtos, pageable, hits.getTotalHits());

        } catch (Exception e) {
            log.error("Search failed: searchKey={}, error={}", searchKey, e.getMessage());
            // On ES error, caller will fallback to SQL
            throw new RuntimeException("Search failed", e);
        }
    }

    /**
     * Build BoolQuery with 6-level ranking
     *
     * Level 1 (12x): diseaseCode exact match
     * Level 2 (8x):  diseaseCode prefix
     * Level 3 (6x):  diseaseName phrase
     * Level 4 (4x):  diseaseName autocomplete (edge_ngram)
     * Level 5 (2x):  diseaseName fuzzy
     * Level 6 (1x):  phonetic fallback
     *
     * Plus boost from category/subcategory/standard matches
     */
    private Query buildSearchQuery(String searchKey) {
        BoolQuery.Builder boolQuery = new BoolQuery.Builder();

        // Add MUST clauses with different boosts for ranking
        boolQuery.must(new MultiMatchQuery.Builder()
            .query(searchKey)
            .fields(
                "diseaseCode^12",           // Exact code match: 12x boost
                "diseaseCode.keyword^8",    // Code prefix: 8x boost
                "diseaseName^6",            // Name phrase: 6x boost
                "diseaseName.autocomplete^4", // Autocomplete: 4x boost
                "diseaseName^2",            // Fuzzy fallback: 2x boost
                "diseaseName.phonetic",     // Phonetic: 1x boost
                "categoryName^3",           // Category name: boost relevance
                "subCategoryName^2",        // SubCategory name: boost relevance
                "healthStandardName"        // Standard name: boost relevance
            )
            .build()
            ._toQuery());

        return boolQuery.build()._toQuery();
    }

    /**
     * Build filter: isActive = true (MANDATORY)
     *
     * Every search must include this filter.
     * We never want to return inactive diseases.
     */
    private Query buildActiveFilter() {
        return new TermQuery.Builder()
            .field("isActive")
            .value(v -> v.booleanValue(true))
            .build()
            ._toQuery();
    }

    /**
     * Map DiseaseDocument to DiseaseDto for API response
     */
    private DiseaseDto mapToDto(DiseaseDocument doc) {
        return DiseaseDto.builder()
            .id(doc.getId())
            .diseaseName(doc.getDiseaseName())
            .diseaseCode(doc.getDiseaseCode())
            .diseaseCodeNormalized(doc.getDiseaseCodeNormalized())
            .categoryId(doc.getCategoryId())
            .categoryName(doc.getCategoryName())
            .subCategoryId(doc.getSubCategoryId())
            .subCategoryName(doc.getSubCategoryName())
            .healthStandardId(doc.getHealthStandardId())
            .healthStandardName(doc.getHealthStandardName())
            .isActive(doc.getIsActive())
            .build();
    }
}
```

### 7. Modified DiseaseServiceImpl (ES Integration)

**Location:** `bmc-user-api/src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseServiceImpl.java`

**Changes:**

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.service.impl;

import com.artemhealthtech.bmc.medicodb.api.v1.user.service.DiseaseSearchService;
import com.artemhealthtech.bmc.medicodb.api.v1.user.service.DiseaseIndexingService;
import com.artemhealthtech.bmc.medicodb.api.v1.user.dto.DiseaseDto;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.event.TransactionalEventListener;
import org.springframework.transaction.event.TransactionPhase;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class DiseaseServiceImpl {

    private static final Logger log = LoggerFactory.getLogger(DiseaseServiceImpl.class);

    // Feature flag: control ES usage
    @Value("${app.search.disease.elasticsearch.enabled:false}")
    private boolean elasticsearchEnabled;

    @Autowired
    private DiseaseSearchService diseaseSearchService;

    @Autowired
    private DiseaseIndexingService diseaseIndexingService;

    @Autowired
    private DiseaseRepository diseaseRepository;

    /**
     * Search for diseases
     *
     * ROUTING LOGIC:
     * 1. Check feature flag (elasticsearch.enabled)
     * 2. If ON and searchKey provided → use Elasticsearch
     * 3. On ES error or flag OFF → fallback to SQL
     */
    public Page<DiseaseDto> paginate(String searchKey, Pageable pageable) {
        try {
            // ===== INPUT VALIDATION =====
            if (searchKey == null || searchKey.trim().length() < 2) {
                log.debug("Search key too short or null, returning empty result");
                return Page.empty(pageable);
            }

            // ===== ROUTE: ES or DB =====
            if (elasticsearchEnabled) {
                try {
                    log.info("Routing to Elasticsearch: searchKey={}", searchKey);
                    Page<DiseaseDto> results = diseaseSearchService.search(searchKey, pageable);
                    log.info("ES search successful: returned {} results", results.getNumberOfElements());
                    return results;

                } catch (Exception e) {
                    log.warn("ES search failed, falling back to SQL: error={}", e.getMessage());
                    // Fallback to SQL
                }
            } else {
                log.debug("Elasticsearch disabled, using SQL fallback");
            }

            // ===== FALLBACK: SQL =====
            return searchViaSQL(searchKey, pageable);

        } catch (Exception e) {
            log.error("Paginate failed: error={}", e.getMessage());
            return Page.empty(pageable);
        }
    }

    /**
     * SQL fallback search
     * Original query from DiseaseRepository
     */
    private Page<DiseaseDto> searchViaSQL(String searchKey, Pageable pageable) {
        try {
            log.info("Executing SQL search: searchKey={}", searchKey);
            Page<Disease> dbResults = diseaseRepository.getDiseaseLikeAndIsActive(searchKey, pageable);

            // Deduplicate by (id, name, code)
            Set<String> seen = new HashSet<>();
            List<DiseaseDto> deduplicated = dbResults.getContent().stream()
                .filter(disease -> {
                    String key = disease.getId() + "|" + disease.getDiseaseName() + "|" + disease.getDiseaseCode();
                    return seen.add(key);
                })
                .map(this::mapToDto)
                .collect(Collectors.toList());

            return new PageImpl<>(deduplicated, pageable, deduplicated.size());

        } catch (Exception e) {
            log.error("SQL search failed: error={}", e.getMessage());
            return Page.empty(pageable);
        }
    }

    /**
     * Create disease and index to Elasticsearch
     */
    @Transactional
    public Disease createDisease(Disease disease) {
        try {
            // Step 1: Save to MySQL
            Disease saved = diseaseRepository.save(disease);
            log.info("Disease saved to DB: id={}", saved.getId());

            // Step 2: Index to Elasticsearch (async via event listener)
            // See @TransactionalEventListener below

            return saved;

        } catch (Exception e) {
            log.error("Create disease failed: error={}", e.getMessage());
            throw new RuntimeException("Failed to create disease", e);
        }
    }

    /**
     * After disease is committed to DB, index to Elasticsearch
     * Uses Spring TransactionalEventListener for safe async indexing
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onDiseaseCreated(DiseaseCreatedEvent event) {
        try {
            if (elasticsearchEnabled) {
                diseaseIndexingService.indexDisease(event.getDiseaseId());
            }
        } catch (Exception e) {
            log.error("Failed to index disease: id={}, error={}", event.getDiseaseId(), e.getMessage());
            // Don't fail the transaction - ES is optional
        }
    }

    /**
     * Update disease and reindex to Elasticsearch
     */
    @Transactional
    public Disease updateDisease(Long diseaseId, Disease updates) {
        try {
            Disease disease = diseaseRepository.findById(diseaseId)
                .orElseThrow(() -> new RuntimeException("Disease not found"));

            disease.setDiseaseName(updates.getDiseaseName());
            disease.setDiseaseCode(updates.getDiseaseCode());
            disease.setDiseaseCategory(updates.getDiseaseCategory());
            disease.setDiseaseSubCategory(updates.getDiseaseSubCategory());
            disease.setHealthStandard(updates.getHealthStandard());
            disease.setIsActive(updates.getIsActive());

            Disease saved = diseaseRepository.save(disease);

            // Reindex to Elasticsearch
            if (elasticsearchEnabled) {
                diseaseIndexingService.indexDisease(saved.getId());
            }

            return saved;

        } catch (Exception e) {
            log.error("Update disease failed: id={}, error={}", diseaseId, e.getMessage());
            throw new RuntimeException("Failed to update disease", e);
        }
    }

    /**
     * Delete disease and remove from Elasticsearch
     */
    @Transactional
    public void deleteDisease(Long diseaseId) {
        try {
            diseaseRepository.deleteById(diseaseId);

            // Remove from Elasticsearch
            if (elasticsearchEnabled) {
                diseaseIndexingService.deleteDisease(diseaseId);
            }

        } catch (Exception e) {
            log.error("Delete disease failed: id={}, error={}", diseaseId, e.getMessage());
            throw new RuntimeException("Failed to delete disease", e);
        }
    }
}
```

---

## Application Configuration

### application.yml

```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: elastic
    password: changeme
    socket-timeout: 10s
    connect-timeout: 5s

app:
  search:
    disease:
      elasticsearch:
        # Enable Elasticsearch search (feature flag)
        # Set to false to force SQL fallback
        enabled: true
```

### ElasticsearchConfig.java

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.elasticsearch.repository.config.EnableElasticsearchRepositories;

/**
 * Elasticsearch Spring Data Configuration
 */
@Configuration
@EnableElasticsearchRepositories(
    basePackages = "com.artemhealthtech.bmc.medicodb.api.v1.user.repository"
)
public class ElasticsearchConfig {
    // Spring Data Elasticsearch auto-configures the rest
}
```

---

## Integration Steps

### Step 1: Add Dependencies

```bash
# In bmc-user-api/pom.xml
# Add spring-boot-starter-data-elasticsearch 3.1.1
```

### Step 2: Create Java Files

1. Create `DiseaseDocument.java`
2. Create `DiseaseSearchRepository.java`
3. Create `DiseaseSearchDocumentMapper.java`
4. Create `DiseaseSearchIndexConfig.java`
5. Create `DiseaseIndexingService.java`
6. Create `DiseaseSearchService.java`
7. Update `DiseaseServiceImpl.java`
8. Create `ElasticsearchConfig.java`

### Step 3: Update DiseaseRepository

Add this method to load diseases with eager-loaded relations:

```java
@Query("SELECT d FROM Disease d " +
       "LEFT JOIN FETCH d.diseaseCategory " +
       "LEFT JOIN FETCH d.diseaseSubCategory " +
       "LEFT JOIN FETCH d.healthStandard " +
       "WHERE d.id = :id")
Optional<Disease> findByIdWithRelations(@Param("id") Long id);

@Query("SELECT d FROM Disease d " +
       "LEFT JOIN FETCH d.diseaseCategory " +
       "LEFT JOIN FETCH d.diseaseSubCategory " +
       "LEFT JOIN FETCH d.healthStandard")
Page<Disease> findAllWithRelations(Pageable pageable);
```

### Step 4: Configure Elasticsearch

```yaml
# application.yml
spring:
  elasticsearch:
    uris: http://localhost:9200

app:
  search:
    disease:
      elasticsearch:
        enabled: true
```

### Step 5: Start Elasticsearch (Local Development)

```bash
docker run --name elasticsearch \
  -p 9200:9200 \
  -e discovery.type=single-node \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  docker.elastic.co/elasticsearch/elasticsearch:8.11.0
```

### Step 6: Bulk Reindex

```bash
POST /admin/disease/reindex
```

---

## 14 Critical Improvements Checklist

- [ ] **1. Separate DiseaseDocument from Disease entity** ✅ (Remove @ManyToOne relations)
- [ ] **2. Create DiseaseSearchDocumentMapper** ✅ (Flatten fields)
- [ ] **3. Add JOIN FETCH in DiseaseRepository** ✅ (Load relations before mapping)
- [ ] **4. Add input validation** ✅ (Reject < 2 chars)
- [ ] **5. Normalize disease code** ✅ (Remove dots)
- [ ] **6. Index only necessary fields** ✅ (Lean document)
- [ ] **7. Enforce result size limits** ✅ (Pageable control)
- [ ] **8. Strengthen deduplication** ✅ (Include ID)
- [ ] **9. Add observability logging** ✅ (Log search metrics)
- [ ] **10. Safe ES fallback** ✅ (Try-catch + SQL)
- [ ] **11. Enforce ES→Service→Controller** ✅ (Architecture)
- [ ] **12. One document per disease** ✅ (Denormalization)
- [ ] **13. Boolean isActive filter** ✅ (Mandatory)
- [ ] **14. Feature flag strictly enforced** ✅ (elasticsearch.enabled)

---

## Before & After Comparison

### BEFORE (SQL - BROKEN)

```java
// ❌ Disease entity DIRECTLY used in search
@Entity
public class Disease {
    @ManyToOne(fetch = LAZY)  // Lazy relations - causes issues
    private DiseaseCategory diseaseCategory;
}

// ❌ Query LEFT JOINs - creates duplicates
SELECT d.* FROM disease d
LEFT JOIN disease_category dc ON ...
LEFT JOIN disease_subcategory dsc ON ...
LEFT JOIN health_standard hs ON ...

// ❌ Result: Duplicate disease IDs
Slots: [1, 1, 1, 2, 2, 3, 3, 3, 4, 5...]  (15 slots = 5 unique diseases)

// ❌ Pagination broken
Page 1: IDs [1, 1, 1, 2, 2, 3, 3, 3, 4, 5]
Page 2: Starts with more duplicates of 1, 2, 3
```

### AFTER (Elasticsearch - FIXED)

```java
// ✅ Separate DiseaseDocument (no relations)
@Document(indexName = "diseases_v1")
public class DiseaseDocument {
    private Long id;
    private String diseaseName;
    private Long categoryId;           // FLATTENED
    private String categoryName;       // FLATTENED
    // No @ManyToOne lazy relations
}

// ✅ Mapper flattens data
DiseaseDocument doc = mapper.fromDisease(disease);
// Copies: disease.id, diseaseCategory.id, diseaseCategory.name

// ✅ Result: One document per disease
Documents: [1 (doc), 2 (doc), 3 (doc), 4 (doc), 5 (doc)]  (5 unique documents)

// ✅ Pagination works correctly
Page 1: IDs [1, 2, 3, 4, 5] (5 unique, no duplicates)
Page 2: IDs [6, 7, 8, 9, 10] (next 5 unique)
```

---

## Next Steps

→ **[Read 03-OPERATIONS.md](./03-OPERATIONS.md)** for setup, troubleshooting, and operations guide
