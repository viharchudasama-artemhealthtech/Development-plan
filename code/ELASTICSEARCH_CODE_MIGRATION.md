# Elasticsearch Code Migration Guide

Date: April 30, 2026

Purpose: show exact files to change and the OLD vs NEW code snippets to migrate Disease search from MySQL-only queries to Elasticsearch-powered search (diseaseName + icdCode). This is a developer-facing checklist + patch guide — follow file-by-file and apply the code changes. The code examples are adapted to Spring Boot 3.1 with Spring Data Elasticsearch.

---

## Summary of changes (high level)

- Add dependencies to `pom.xml` for Spring Data Elasticsearch and the Elasticsearch Java client.
- Add ES configuration bean(s).
- Add ES document model `DiseaseSearchDocument`.
- Add Spring Data ES repository `DiseaseSearchRepository`.
- Add `DiseaseMapper` to map JPA entity → ES document.
- Add `IndexingService` to index single disease and bulk reindex.
- Add `DiseaseSearchService` + `QueryBuilder` to construct ES queries supporting fuzziness, phonetic, and autocomplete.
- Modify `DiseaseServiceImpl` to prefer ES when enabled and fallback to DB when ES fails.
- Add controller endpoints for autocomplete and admin reindex.
- Add index mapping JSON file and `application-*.yml` properties.

---

> Note: replace placeholders like `com.artemhealthtech.bmc.medicodb.api.v1.user.elasticsearch` with your preferred package names.

## 1) Add dependencies (pom.xml)

File: pom.xml

// OLD CODE

```xml
<!-- existing dependencies section (no ES) -->
```

// NEW CODE (add these dependencies inside `<dependencies>`)

```xml
<!-- Spring Data Elasticsearch -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>

<!-- Elasticsearch Java API client (optional - for advanced operations) -->
<dependency>
  <groupId>co.elastic.clients</groupId>
  <artifactId>elasticsearch-java</artifactId>
  <version>8.11.0</version>
</dependency>

<!-- Testcontainers elasticsearch for integration tests (test scope) -->
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>elasticsearch</artifactId>
  <scope>test</scope>
</dependency>
```

Notes:

- After adding, run `mvn -U clean package` to refresh dependencies.

---

## 2) Add application properties

File: src/main/resources/application-dev.yml (and application.yml / prod)

// OLD CODE

```yaml
# (no elasticsearch settings)
```

// NEW CODE

```yaml
spring:
  data:
    elasticsearch:
      client:
        endpoints: ${ELASTICSEARCH_URIS:http://localhost:9200}

app:
  search:
    elasticsearch:
      enabled: true
      disease:
        index-name: disease_v1
        create-index-on-startup: true
```

Notes:

- `ELASTICSEARCH_URIS` can be provided as env var.
- Keep `app.search.elasticsearch.enabled` as feature flag for safe rollout.

---

## 3) Create Elasticsearch config bean

File: `src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/elasticsearch/config/ElasticsearchConfig.java`

// NEW CODE

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.elasticsearch.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestClientBuilder;
import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.transport.rest_client.RestClientTransport;
import co.elastic.clients.json.jackson.JacksonJsonpMapper;
import co.elastic.clients.transport.Transport;
import org.apache.http.HttpHost;

@Configuration
public class ElasticsearchConfig {

    @Value("${spring.data.elasticsearch.client.endpoints}")
    private String esEndpoints;

    @Bean
    public RestClient restClient() {
      // single endpoint expected: http://localhost:9200
      HttpHost httpHost = HttpHost.create(esEndpoints);
      RestClientBuilder builder = RestClient.builder(httpHost);
      return builder.build();
    }

    @Bean
    public ElasticsearchClient elasticsearchClient(RestClient restClient) {
      Transport transport = new RestClientTransport(restClient, new JacksonJsonpMapper());
      return new ElasticsearchClient(transport);
    }
}
```

Notes:

- This provides low-level `ElasticsearchClient` for admin operations (index creation) and `RestClient` for the transport.
- Optionally add `ElasticsearchOperations` bean if you prefer Spring Data templates.

---

## 4) Add index mapping JSON file

File: `src/main/resources/elasticsearch/disease-index-mapping.json`

// NEW FILE (full mapping + analyzers)

```json
{
  "settings": {
    "analysis": {
      "filter": {
        "edge_ngram_filter": {
          "type": "edge_ngram",
          "min_gram": 3,
          "max_gram": 15,
          "side": "front"
        },
        "phonetic_filter": {
          "type": "phonetic",
          "encoder": "double_metaphone",
          "replace": true
        },
        "code_normalizer": {
          "type": "pattern_replace",
          "pattern": "\\.",
          "replacement": ""
        }
      },
      "analyzer": {
        "disease_search_analyzer": {
          "tokenizer": "standard",
          "filter": ["lowercase", "word_delimiter_graph", "stop", "snowball"]
        },
        "disease_autocomplete_index_analyzer": {
          "tokenizer": "standard",
          "filter": ["lowercase", "edge_ngram_filter"]
        },
        "disease_autocomplete_search_analyzer": {
          "tokenizer": "standard",
          "filter": ["lowercase"]
        },
        "disease_phonetic_analyzer": {
          "tokenizer": "standard",
          "filter": ["lowercase", "phonetic_filter"]
        },
        "code_analyzer": {
          "tokenizer": "keyword",
          "filter": ["lowercase", "code_normalizer"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "id": { "type": "keyword" },
      "diseaseName": {
        "type": "search_as_you_type",
        "analyzer": "disease_search_analyzer",
        "search_analyzer": "disease_autocomplete_search_analyzer",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 },
          "autocomplete": {
            "type": "text",
            "analyzer": "disease_autocomplete_index_analyzer",
            "search_analyzer": "disease_autocomplete_search_analyzer"
          },
          "phonetic": {
            "type": "text",
            "analyzer": "disease_phonetic_analyzer"
          }
        }
      },
      "icdCode": {
        "type": "text",
        "analyzer": "code_analyzer",
        "fields": {
          "keyword": { "type": "keyword" },
          "autocomplete": {
            "type": "text",
            "analyzer": "disease_autocomplete_index_analyzer",
            "search_analyzer": "disease_autocomplete_search_analyzer"
          }
        }
      },
      "icdCodeNormalized": { "type": "keyword" },
      "isActive": { "type": "boolean" }
    }
  }
}
```

Notes:

- This mapping supports autocomplete, phonetic search, and ICD code normalization.

---

## 5) Create ES document model

File: `src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/elasticsearch/document/DiseaseSearchDocument.java`

// NEW CODE

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.elasticsearch.document;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.Field;
import org.springframework.data.elasticsearch.annotations.FieldType;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Document(indexName = "disease_v1")
public class DiseaseSearchDocument {

    @Id
    private String id;

    @Field(type = FieldType.Text, name = "diseaseName")
    private String diseaseName;

    @Field(type = FieldType.Text, name = "icdCode")
    private String icdCode;

    @Field(type = FieldType.Keyword, name = "icdCodeNormalized")
    private String icdCodeNormalized;

    @Field(type = FieldType.Boolean)
    private Boolean isActive;

}
```

Notes:

- Keep document minimal (only fields required for searchable surface). Additional fields (category, subcategory) can be added later.

---

## 6) Create Spring Data Elasticsearch repository

File: `src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/elasticsearch/repository/DiseaseSearchRepository.java`

// NEW CODE

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.elasticsearch.repository;

import com.artemhealthtech.bmc.medicodb.api.v1.user.elasticsearch.document.DiseaseSearchDocument;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface DiseaseSearchRepository extends ElasticsearchRepository<DiseaseSearchDocument, String> {
}
```

---

## 7) Create mapper: JPA entity → ES document

File: `src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/elasticsearch/mapper/DiseaseMapper.java`

// NEW CODE

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.elasticsearch.mapper;

import com.artemhealthtech.bmc.medicodb.api.v1.user.elasticsearch.document.DiseaseSearchDocument;
import com.hospisoft.model.Disease;
import org.springframework.stereotype.Component;

@Component
public class DiseaseMapper {

    public DiseaseSearchDocument toDocument(Disease disease) {
        if (disease == null) return null;
        String normalized = null;
        if (disease.getDiseaseCode() != null) {
            normalized = disease.getDiseaseCode().replaceAll("\\.", "").toLowerCase();
        }
        return DiseaseSearchDocument.builder()
                .id(String.valueOf(disease.getId()))
                .diseaseName(disease.getDiseaseName())
                .icdCode(disease.getDiseaseCode())
                .icdCodeNormalized(normalized)
                .isActive(disease.getIsActive())
                .build();
    }
}
```

---

## 8) Indexing service (single + bulk)

File: `src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/elasticsearch/service/IndexingService.java`

// NEW CODE

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.elasticsearch.service;

import com.artemhealthtech.bmc.medicodb.api.v1.user.elasticsearch.document.DiseaseSearchDocument;
import com.artemhealthtech.bmc.medicodb.api.v1.user.elasticsearch.mapper.DiseaseMapper;
import com.artemhealthtech.bmc.medicodb.api.v1.user.elasticsearch.repository.DiseaseSearchRepository;
import com.hospisoft.model.Disease;
import com.artemhealthtech.bmc.medicodb.api.v1.user.repository.DiseaseRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Service;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.ArrayList;
import java.util.List;

@Service
public class IndexingService {

    private static final Logger log = LoggerFactory.getLogger(IndexingService.class);

    @Autowired
    private DiseaseSearchRepository searchRepository;

    @Autowired
    private DiseaseRepository diseaseRepository;

    @Autowired
    private DiseaseMapper mapper;

    public void indexDisease(Long diseaseId) {
        try {
            Disease d = diseaseRepository.findById(diseaseId).orElse(null);
            if (d == null) return;
            DiseaseSearchDocument doc = mapper.toDocument(d);
            searchRepository.save(doc);
            log.info("Indexed disease id={}", diseaseId);
        } catch (Exception e) {
            log.error("Failed to index disease {}: {}", diseaseId, e.getMessage(), e);
        }
    }

    public void reindexAll() {
        int page = 0;
        int size = 500;
        while (true) {
            var p = diseaseRepository.findAll(PageRequest.of(page, size));
            var list = p.getContent();
            if (list.isEmpty()) break;
            List<DiseaseSearchDocument> docs = new ArrayList<>();
            for (Disease d : list) docs.add(mapper.toDocument(d));
            searchRepository.saveAll(docs);
            page++;
            if (!p.hasNext()) break;
        }
    }

    public void deleteFromIndex(Long diseaseId) {
        searchRepository.deleteById(String.valueOf(diseaseId));
    }
}
```

Notes:

- Consider making `indexDisease` asynchronous (e.g., `@Async`) for non-blocking updates.

---

## 9) Query builder + DiseaseSearchService (search logic)

File: `src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/elasticsearch/service/DiseaseSearchService.java`

// NEW CODE (simplified using `ElasticsearchOperations`)

```java
package com.artemhealthtech.bmc.medicodb.api.v1.user.elasticsearch.service;

import com.artemhealthtech.bmc.medicodb.api.v1.user.elasticsearch.document.DiseaseSearchDocument;
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
import co.elastic.clients.elasticsearch._types.query_dsl.Query;
import org.elasticsearch.index.query.MultiMatchQueryBuilder;
import org.elasticsearch.index.query.QueryBuilders;

import java.util.List;
import java.util.stream.Collectors;

@Service
public class DiseaseSearchService {

    @Autowired
    private ElasticsearchOperations elasticsearchOperations;

    public Page<DiseaseDto> search(String searchKey, Pageable pageable) {
        if (searchKey == null || searchKey.trim().length() < 2) {
            return new PageImpl<>(List.of(), pageable, 0);
        }
        String q = searchKey.trim();

        // Build native query using a six-level boolean ranking described in the migration guide.
        // This implements exact matches, phrase matches, autocomplete/prefix, fuzzy, phonetic, and fallback fuzzy.
        String normalized = q.replaceAll("\\\\.", "").toLowerCase();

        org.elasticsearch.index.query.BoolQueryBuilder topLevel = org.elasticsearch.index.query.QueryBuilders.boolQuery()
          // 1) Exact term matches (highest boost)
          .should(org.elasticsearch.index.query.QueryBuilders.boolQuery()
            .should(org.elasticsearch.index.query.QueryBuilders.termQuery("diseaseName.keyword", q).boost(12f))
            .should(org.elasticsearch.index.query.QueryBuilders.termQuery("icdCode.keyword", q).boost(12f)))
          // 2) Phrase / exact normalized code (high boost)
          .should(org.elasticsearch.index.query.QueryBuilders.boolQuery()
            .should(org.elasticsearch.index.query.QueryBuilders.matchPhraseQuery("diseaseName", q).boost(8f))
            .should(org.elasticsearch.index.query.QueryBuilders.termQuery("icdCodeNormalized", normalized).boost(8f)))
          // 3) Autocomplete / prefix matches (mid boost)
          .should(org.elasticsearch.index.query.QueryBuilders.boolQuery()
            .should(org.elasticsearch.index.query.QueryBuilders.prefixQuery("diseaseName.autocomplete", q).boost(6f))
            .should(org.elasticsearch.index.query.QueryBuilders.prefixQuery("icdCode.autocomplete", normalized).boost(6f)))
          // 4) Fuzzy match with automatic fuzziness (lower boost)
          .should(org.elasticsearch.index.query.QueryBuilders.boolQuery()
            .should(org.elasticsearch.index.query.QueryBuilders.matchQuery("diseaseName", q).fuzziness(org.elasticsearch.common.unit.Fuzziness.AUTO).boost(4f))
            .should(org.elasticsearch.index.query.QueryBuilders.matchQuery("icdCode", q).fuzziness(org.elasticsearch.common.unit.Fuzziness.AUTO).boost(4f)))
          // 5) Phonetic matches and code matches (lower boost)
          .should(org.elasticsearch.index.query.QueryBuilders.boolQuery()
            .should(org.elasticsearch.index.query.QueryBuilders.matchQuery("diseaseName.phonetic", q).boost(2f))
            .should(org.elasticsearch.index.query.QueryBuilders.matchQuery("icdCode", q).boost(2f)))
          // 6) Broad fuzzy fallback (lowest boost)
          .should(org.elasticsearch.index.query.QueryBuilders.boolQuery()
            .should(org.elasticsearch.index.query.QueryBuilders.matchQuery("diseaseName", q).fuzziness(org.elasticsearch.common.unit.Fuzziness.AUTO).boost(1f))
            .should(org.elasticsearch.index.query.QueryBuilders.matchQuery("icdCode", q).fuzziness(org.elasticsearch.common.unit.Fuzziness.AUTO).boost(1f)));

        // Apply filter for active diseases and require at least one should clause to match.
        topLevel.minimumShouldMatch(1);
        org.elasticsearch.index.query.BoolQueryBuilder finalQuery = org.elasticsearch.index.query.QueryBuilders.boolQuery()
          .must(topLevel)
          .filter(org.elasticsearch.index.query.QueryBuilders.termQuery("isActive", true));

        NativeSearchQuery query = new NativeSearchQueryBuilder()
          .withQuery(finalQuery)
          .withPageable(pageable)
          .build();

        SearchHits<DiseaseSearchDocument> hits = elasticsearchOperations.search(query, DiseaseSearchDocument.class);
        List<DiseaseDto> dtos = hits.stream().map(hit -> {
            var d = hit.getContent();
            DiseaseDto dto = new DiseaseDto();
            dto.setId(Long.valueOf(d.getId()));
            dto.setDiseaseName(d.getDiseaseName());
            dto.setDiseaseCode(d.getIcdCode());
            dto.setRelevanceScore(hit.getScore());
            return dto;
        }).collect(Collectors.toList());

        long total = hits.getTotalHits();
        return new PageImpl<>(dtos, pageable, total);
    }
}
```

Notes:

- This uses ElasticsearchOperations and native query builders.
- `fuzziness` and multi-field boosts are applied.
- For phonetic matching we rely on the `disease_phonetic_analyzer` defined in mapping.

---

## 10) Modify `DiseaseServiceImpl` to route to ES with fallback

File: `src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/service/impl/DiseaseServiceImpl.java`

// OLD CODE snippet (existing paging + SQL search)

```java
if (ObjectUtils.isEmpty(paginationRequestDTO.getSearchKey())) {
    diseases = diseaseRepository.findByIsActive(paginationRequestDTO.getIsActive(), pageable);
} else {
    // builds containsPattern, startsWith etc
    diseases = diseaseRepository.getDiseaseLikeAndIsActive(..., pageable);
}

Set<String> seen = new HashSet<>();
List<DiseaseDto> uniqueList = diseases.stream()
    .map(this::modelToDto)
    .filter(dto -> seen.add((dto.getDiseaseName() + "|" + dto.getDiseaseCode()).toLowerCase()))
    .toList();

return new PageImpl<>(uniqueList, diseases.getPageable(), diseases.getTotalElements());
```

// NEW CODE snippet (use ES with fallback)

```java
@Autowired
private DiseaseSearchService diseaseSearchService; // new ES service

@Override
public Page<DiseaseDto> paginate(PaginationRequestDTO paginationRequestDTO) {
    Pageable pageable = PageRequest.of(paginationRequestDTO.getPage(), paginationRequestDTO.getSize());

    if (ObjectUtils.isEmpty(paginationRequestDTO.getSearchKey())) {
        // same as before when no searchKey
        Page<Disease> diseases = diseaseRepository.findByIsActive(paginationRequestDTO.getIsActive(), pageable);
        List<DiseaseDto> dtos = diseases.stream().map(this::modelToDto).toList();
        return new PageImpl<>(dtos, pageable, diseases.getTotalElements());
    }

    // Prefer Elasticsearch
    try {
        if (appProperties.getSearch().getElasticsearch().isEnabled()) {
            Page<DiseaseDto> esPage = diseaseSearchService.search(paginationRequestDTO.getSearchKey(), pageable);
            return esPage;
        }
    } catch (Exception e) {
        log.warn("Elasticsearch search failed, falling back to DB", e);
        // fall through to DB fallback
    }

    // DB fallback (existing logic) with dedup
    Page<Disease> diseases = diseaseRepository.getDiseaseLikeAndIsActive(..., pageable);
    Set<String> seen = new HashSet<>();
    List<DiseaseDto> uniqueList = diseases.stream()
            .map(this::modelToDto)
            .filter(dto -> seen.add((dto.getDiseaseName() + "|" + dto.getDiseaseCode()).toLowerCase()))
            .toList();
    return new PageImpl<>(uniqueList, diseases.getPageable(), diseases.getTotalElements());
}
```

Notes:

- `appProperties` is a configuration wrapper for `app.search.elasticsearch.enabled`.
- Keep the SQL fallback in place to ensure reliability during rollout.

---

## 11) Controller changes (optional new endpoints)

File: `src/main/java/com/artemhealthtech/bmc/medicodb/api/v1/user/controller/DiseaseController.java`

// NEW endpoints to add

```java
@GetMapping("/autocomplete")
public ResponseEntity<ResponseDto> autocomplete(@RequestParam String q, @RequestParam(defaultValue = "5") int size) {
    List<DiseaseDto> suggestions = diseaseSearchController.autocomplete(q, size);
    return ResponseEntity.ok(CollectionResponseDto.builder()
            .responseCode(HttpStatus.OK.value())
            .responseObject(suggestions)
            .totalRecords((long) suggestions.size())
            .build());
}

@PostMapping("/admin/reindex")
public ResponseEntity<ResponseDto> reindexAll() {
    indexingService.reindexAll();
    return ResponseEntity.ok(ResponseDto.builder().responseCode(HttpStatus.OK.value()).responseMessage("Reindex started").build());
}
```

Notes:

- These are additive endpoints that aid dev & ops.

---

## 12) Tests & Validation

- Unit test `DiseaseMapper` mapping.
- Integration tests for `DiseaseSearchService` using Testcontainers Elasticsearch.
- End-to-end test: reindex, then call `POST /api/v1/user/disease/page` and verify results returned and relevance scores.

---

## 13) Deployment & Rollout

1. Deploy code with ES disabled (`app.search.elasticsearch.enabled=false`).
2. Start ES cluster and create index using mapping JSON (or let the app create it at startup if configured).
3. Run `POST /admin/disease/reindex` to populate ES.
4. Enable ES feature flag for 10% canary users.
5. Monitor fallbacks & performance, then ramp up.

Alias & zero-downtime index swap:

- Use versioned indices and a stable alias: create indices like `disease_v1`, `disease_v2` and point the application to alias `disease_search`.
- Create index with mapping then `POST /_aliases` to atomically switch alias to new index when ready:

  Example (create new index then switch alias):
  1. Create `disease_v2` with mapping JSON.
  2. `POST /_aliases` payload:

  ```json
  {
    "actions": [
      { "add": { "index": "disease_v2", "alias": "disease_search" } },
      { "remove": { "index": "disease_v1", "alias": "disease_search" } }
    ]
  }
  ```

- The application (and `ElasticsearchOperations` / `ElasticsearchClient` usage) should always read/write using the alias `disease_search` so swaps are transparent.

Operational notes:

- Keep DB fallback: the service layer should try ES first and fall back to the DB on exceptions or timeout to guarantee availability.
- Add an admin health endpoint to check alias existence and index document counts before enabling the feature flag.
- Highlighting & frontend UX: return `highlight` fragments from ES for the `diseaseName` field so the frontend can emphasize matched tokens. For autocomplete, return only `diseaseName` and `icdCode` with minimal payload.
- Frontend suggestion: debounce 250–350ms on input, enforce min characters=3 for autocomplete to avoid load (matches ngram min_gram=3).
- Scoring rationale: the explicit boosts (12,8,6,4,2,1) were chosen to strongly prefer exact canonical matches, then phrase/code matches, then prefix/autocomplete, then fuzzy, then phonetic, then broad fuzzy fallback; tune in production with user telemetry (A/B tests) and query logs.

---

## 14) Patch Hints (how to apply changes)

- Use `git checkout -b feat/elasticsearch-disease-search`.
- Apply changes file-by-file using small commits: `git add <file>` / `git commit -m "Add DiseaseSearchDocument"`.
- Run `mvn -DskipTests=false test` and fix compile errors.
- Run integration tests with Testcontainers.

---

If you want, I can now:

- apply these exact edits into the repository (generate Java files and patches), or
- produce a smaller PR-ready patch set with `apply_patch` for each change.

Which would you prefer? (I will not modify the codebase until you confirm.)
