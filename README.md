

## OpenSearch Ingestion Flow (High-Level)

**Goal:**
Ingest images + metadata into OpenSearch so they become searchable.

### 1. Ingestion Service (OpenSearch Ingest)

* The ingest service pushes data into OpenSearch.
* Contributors upload:

  * Images
  * Image metadata
* Write operations happen to DynamoDB tables:

**Main DynamoDB tables:**

* `product_metadata` – product info
* `image_metadata` – image info
* `contributors` – contributor details
* `curation` – categories / curation info

---

### 2. Event Notification (Lambda + SNS)

* A Lambda function runs when new data is ingested.
* Lambda publishes an event to **SNS** (publish/subscribe system).
* SNS publishes messages to a **Topic**.
* Consumers (queues) subscribed to this topic receive notifications.

---

### 3. Queue Processing (SQS)

* SQS receives messages about newly ingested images.
* Consumers poll SQS in **FIFO** manner.
* The message contains image IDs / product IDs to process further.

---

### 4. Product API Fetch

* A service calls **Product API** to fetch remaining data.
* Additional metadata is fetched from **DynamoDB**.
* All data is combined into a final document.

---

### 5. Transform & Index

* Data is transformed into the OpenSearch schema.
* Final transformed data is indexed into OpenSearch.
* Once indexed → it becomes searchable.

---

## OpenSearch Cluster & Indexing

### In OpenSearch Cluster:

* Ingested data becomes **indexed data**.
* There are **two clusters**:

  * Bot domain
  * User domain

Each cluster contains indexes where queries are sent.

---

## Querying in OpenSearch

* We use a **Search API** (wrapper service) instead of querying OpenSearch directly.
* OpenSearch queries are not very user-friendly, so APIs handle this.

---

## How Search Works Internally (Lucene Basics)

OpenSearch uses **Lucene inverted index**.

Instead of scanning every document:

* Each word points to the documents that contain it.

Example:

```
"cat"   → doc2, doc7
"dog"   → doc3
"mountain" → doc9
```

So searching is fast because OpenSearch jumps directly to relevant documents.

---

## Tokenization & Stop Words

When indexing text:

* Stop words are removed first (e.g., "is", "the", "and")
* Text is tokenized into searchable terms

Example:

```
"The dog is on the mountain"
→ tokens: dog, mountain
```

---

## Mappings & Schema

* Mappings define how each field is indexed.
* Mapping = schema for OpenSearch.
* Important field types:

| Type                       | Use case                              |
| -------------------------- | ------------------------------------- |
| `text`                     | Full-text search                      |
| `keyword`                  | Exact match, filtering, aggregations  |
| `integer`, `long`, `float` | Numeric search                        |
| `boolean`                  | True/false                            |
| `date`                     | Date filtering                        |
| `knn_vector`               | Vector embeddings for semantic search |

If a field is:

* `text` or `keyword` and indexed → it is searchable
* Not indexed → it cannot be searched

---

## Shards (How Data is Stored)

* Index is split into **shards** (partitions).
* Shards allow:

  * Parallel search
  * Scalability
  * Faster queries

---

## Vector Search (KNN / Embeddings)

* Used for semantic search (image/text similarity)
* Example:

  * “Dog on mountain” → converted to embedding
  * Stored as `knn_vector`
* OpenSearch uses vector engines (e.g., FAISS or NMSLIB internally)
* KNN finds nearest vectors based on similarity

---

## OpenSearch Index Template

Index templates define:

* Settings
* Mappings
* Shards
* KNN configuration

Templates ensure all new indexes follow the same structure.

---

## When Is a Field Searchable?

A field is searchable in OpenSearch if:

* It is indexed
* It has a searchable type (`text`, `keyword`, `numeric`, `boolean`, `date`, `knn_vector`)

If it’s stored but **not indexed**, it can be returned in results but not searched. ie index is false 



---

## Summary (Plain English)

* Data flows: **Ingest → SNS → SQS → Product API + DynamoDB → Transform → OpenSearch**
* OpenSearch indexes data using **Lucene inverted index**
* Text is tokenized and stop words removed
* Schema (mappings) decides what is searchable
* Vectors enable semantic search
* Shards help with scale and speed
* Index templates keep things consistent

## Local Setup for Testing OpenSearch

### Docker Compose (Local Environment)

* `docker-compose.yml` is used to run everything locally.
* It sets up:

  * OpenSearch
  * DynamoDB (local)
  * Other dependent services
* This helps developers test without using real AWS services.

**How to start locally:**

cd ../opensearch-ingest/local-dev

```bash
docker compose up -d
---

## Terraform (Mocking the Real Setup)

* Terraform config exists for cloud infra.
* Locally, we **simulate parts of the real setup**:

  * Domains
  * Index names
  * Aliases
* This helps keep local and cloud environments similar.

---

## Creating an Index (Local Test)

* We create indexes using curl:

```bash
./scripts/create-index
http://localhost:9200/<index-name>
```

Example from notes:

```bash
http://localhost:9200/500k-384d-titan-mm-v1
```

**Meaning:**

* `localhost:9200` → OpenSearch domain (local)
* `500k-384d-titan-mm-v1` → index name
* `v1_0_video_lexical` -> Another index name 

  * `v1` = old index version
  * `v2` = new index version

---

## Aliases (Why v1 & v2 Both Work)

* Aliases let multiple index versions be searched using **one name**.
* The Search API points to the alias, not the raw index.

Example:

```
alias: product-search
→ points to:
   - 500k-384d-titan-mm-v1
   - 500k-384d-titan-mm-v2
```

So:

* You can reindex into `v2`
* Search still works on both v1 and v2
* No client changes needed

**Rule:**
Aliases must be in the **same OpenSearch cluster**.

---
## Local Testing Setup (Open Cluster)

### Seeding Local Test Data

We use the **test-packs repo** to seed data into OpenSearch:

```bash
./scripts/packs.ts seed packs/search-api.ts
```

This:

* Inserts local test documents into OpenSearch cluster
* Lets you test search flows without production data

---
Q : Why not use ./scripts/packs.ts seed packs/opensearch-ingest.ts instead of search-api.ts? : 
Because Integration testing  has to be done as the opensearch-ingest.ts pushes data to product-api container 
### Testing Queries (Test Packs)

* Go to **test-packs**
* Pick any random ingested document ID eg : 
* Use that ID to test search results

Example:

```
Pick a random doc ID → run search → verify it appears in results ( Go to querying to understand how it works ) 
```

## Search API (How Queries Are Sent)

awsume dev (for getting permission for aws operation in hybrid search)

npm run dev 

{Go to querying for direct testing }

##understanding how search api helps in workflow 

We do NOT query OpenSearch directly from frontend.

Instead:

```
Client → Search API → OpenSearch
```

* The Search API:

  * Receives user query
  * Validates inputs
  * Builds OpenSearch query
  * Calls OpenSearch
  * Cleans response before returning it

Example endpoint:

```bash
http://localhost:3002/v2/domains/<domain>/search
```

---

## v1 vs v2 Search API

* v1 and v2 are **different index versions**
* Both can be searched using the same API because:

  * The alias points to both indexes
* Search API supports both

---

## What the Search API Does Internally

1. Controller receives request
2. Input params are validated
3. Query is built using OpenSearch DSL
4. OpenSearch service is called
5. Response is cleaned up
6. Final response is returned to client

---

## Controlling What Comes Back (Source Filtering)

* You can control which fields are returned.
* Example:

  * Don’t return large vectors
  * Don’t return internal fields

This keeps responses small and fast.

---

## Query Builder (Utility Layer)

* A helper layer converts user params → OpenSearch query
* Example:

  * keywords
  * filters
  * pagination
  * hybrid search logic
* Keeps controllers clean and readable.

---

## Search Types (One-Line Mental Model)

* **Lexical search** → finds text matches (exact words)
* **KNN search** → finds meaning (semantic similarity)
* **Hybrid search** → combines both and blends scores

So:

```
Lexical = keyword match  
KNN = meaning match  
Hybrid = best of both
```

---

## Default Search Fields

* The system defines default fields to search in:

  * title
  * description
  * tags
* If the user searches “dog”, we search in these fields by default.

---

## Summary (Plain English)

* Docker Compose = local OpenSearch testing
* Terraform = mirrors real cloud setup
* v1 & v2 are index versions
* Aliases let both versions work without breaking search
* Search API builds OpenSearch queries for us
* Hybrid search = keyword + meaning
* Source filtering keeps responses small

--

# Search API → OpenSearch (Testing + Concepts)
## How a Search Request Flows (Code-Level View)

**High-level flow:**

Client
  → controller.ts
      → transforms request params
  → query.utils.ts
      → transformQuery()
      → add pagination / diversification
  → opensearch.service.ts
      → buildQuery()
  → OpenSearch cluster
  → response cleaned (_source filtering)
  → Client

### 1. Controller (controller.ts)

* Receives request params from client
* Transforms params into internal format
* Does basic validation
* Passes clean params to OpenSearch service

---

### 2. Query Builder (opensearch.service.ts)

* Builds OpenSearch query using:

  * `buildQuery()` function
* Decides:

  * lexical / knn / hybrid
  * filters
  * pagination
* Sends final query to OpenSearch

---

### 3. Query Utils (query.utils.ts)

* Helper functions to transform query parts
* Keeps query-building logic clean
* Same function name exists for transforming OpenSearch queries

---

### 4. Constants (constants.ts)

* Contains default search fields
* Example:

  * title
  * description
  * tags
* These fields are used when building queries
* So when user searches `"dog"`, we search inside these fields

---

### 5. Controlling the Response (_source)

* `_source` decides **what fields come back**
* If we don’t need vector data → don’t include it in `_source`
* This:

  * reduces payload size
  * improves response speed
* Since the cluster is open now, vector fields are skipped for now

---

### Pagination Defaults

* Default page size = **10**
* Total results depend on:

  * page size
  * page number
* If not provided, Search API uses defaults
querying in v2 is in camel case 

---

### Searching by Sequence / ID

* If you include a `sequence` or ID in query:

  * Search API passes it
  * OpenSearch filters on it
* Useful for debugging exact documents

---

## OpenSearch Basics (Search Logic)

### Relevance (Keyword Search)

* Keyword search ranks results by **relevance**
* Modern OpenSearch uses **BM25** scoring (improved TF-IDF)
* Higher score = more relevant

Plain meaning:

* If your search words appear more meaningfully in a document,
  that document ranks higher.

---

### Vector Search (Meaning Search)

* Converts text/images into numbers (embeddings)
* Finds nearest matches in vector space
* Good for “meaning” similarity

---

### Hybrid Search (Best of Both)

* Keyword search → exact matches
* Vector search → meaning matches
* Hybrid → blends both scores

Plain mental model:

```
Keyword = exact word match  
Vector  = meaning match  
Hybrid  = combine both
```

---

## OpenSearch Core Concepts (Quick Recap)

### Indexing & Shards

* Once data is indexed:

  * It is split into **shards**
* Shards:

  * are subsets of documents
  * are spread across nodes
  * improve speed and scale

```
Cluster → Nodes → Index → Shards → Documents
```

---

### Inverted Index (How Text Search Is Fast)

* OpenSearch stores:

  * word → list of document IDs
* So search jumps directly to matching docs
* Text is tokenized during indexing

---

### Analyzer (How Text Is Processed)

* Defined in index settings:

  * `index.settings.analysis`
* Analyzer:

  * removes stop words
  * tokenizes text
* Example:

```
"The dog is on the mountain"
→ dog, mountain
```

---

## Summary (Plain English)

* `controller.ts` cleans request params
* `opensearch.service.ts` builds query
* `query.utils.ts` helps transform queries
* `constants.ts` defines searchable fields
* `_source` controls what data comes back
* Local test data is seeded using test-packs
* BM25 handles keyword relevance
* Vector search handles meaning
* Hybrid combines both
* Shards split data for speed
* Inverted index makes search fast

---

## One-Line Mental Model

```
Keyword search = word match  
Vector search  = meaning match  
Hybrid         = smart blend of both
```


---

## Querying for Testing

`GET http://localhost:9200/500k_384d_titan_mm_v1/_search`

### Lexical Query

```json
{
    "query" :{
        "match": {
            "Ref": "2F8F7J6"
        }
    }
}

```

### Normal Query

```json
{
  "query": {
    "match": {
      "Caption": "product"
    }
  }
}

```

### Normal search with case insensitivity and range

```json
{
  "query": {
    "wildcard": {
      "Caption": {
        "value": "*product*",
        "case_insensitive": true
      }
    }
  }
}

```

### Search with multiple fields

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "AssetStatus": "InCatalog" } },
        { "term": { "Orientation": "Landscape" } }
      ]
    }
  }
}

```

### LexicalSearch Result Example

```json
{
  "openSearchQuery": {
    "_source": {
      "include": [
        "Sequence", "Ref", "Id", "PseudoId", "Pseudonym", "CompanyName", "Caption",
        "CaptionDe", "CaptionFr", "CaptionIt", "LicenseType", "ModelRelease",
        "PropertyRelease", "DateTaken", "UploadDate", "PixelsX", "PixelsY",
        "IsEditorialOnly", "AssetType", "Dataco", "IsNews", "SearchCollectionId",
        "ImageBoost", "Sample", "Duration", "UserId"
      ]
    },
    "size": 500,
    "from": 0,
    "track_total_hits": false,
    "query": {
      "function_score": {
        "query": {
          "bool": {
            "must": [
              {
                "query_string": {
                  "fields": ["EssentialWords^5", "Caption^3", "MainWords", "ExtraWords", "Ref"],
                  "query": "golden retriever",
                  "default_operator": "AND",
                  "type": "cross_fields"
                }
              }
            ],
            "filter": {
              "bool": {
                "filter": [
                  { "terms": { "AssetStatus": ["InCatalog", "AwaitingSubmission"] } }
                ]
              }
            }
          }
        },
        "functions": [
          { "field_value_factor": { "field": "PseudonymRank", "missing": 48 } },
          { "filter": { "term": { "ImageBoost": 5 } }, "weight": 10 },
          { "filter": { "term": { "ImageBoost": -5 } }, "weight": 0.1 },
          { "filter": { "term": { "IsArchive": true } }, "weight": 0.2 },
          { "filter": { "term": { "BlackAndWhite": true } }, "weight": 0.2 },
          { "filter": { "terms": { "AssetType": ["illustration"] } }, "weight": 0.2 },
          { "filter": { "terms": { "AssetType": ["vector"] } }, "weight": 0.2 }
        ],
        "score_mode": "multiply",
        "boost_mode": "multiply"
      }
    },
    "aggs": {
      "IsACutOut": { "terms": { "field": "IsACutOut", "min_doc_count": 1 } },
      "AssetType": { "terms": { "field": "AssetType", "min_doc_count": 1 } },
      "BlackAndWhite": { "terms": { "field": "BlackAndWhite", "min_doc_count": 1 } }
    }
  }
}

```

### Combine filter + KNN

```json
{
  "size": 10,
  "query": {
    "bool": {
      "filter": [
        { "term": { "AssetStatus": "InCatalog" } },
        { "term": { "AssetType": "photograph" } }
      ],
      "must": {
        "knn": {
          "Vector": {
            "vector": [ /* full 384 values here */ ],
            "k": 10
          }
        }
      }
    }
  }
}

```

### Knn Query

```json
{
  "size": 10,
  "query": {
    "knn": {
      "Vector": {
        "vector": [ /* paste the full 384-dim vector here */ ],
        "k": 10
      }
    }
  }
}

```

### Match all

```json
{
  "size": 100,
  "query": {
    "match_all": {}
  }
}

```

---

## Mapping of the Query

Obtained via `GET http://localhost:9200/500k_384d_titan_mm_v1/_mapping`

```json
{
  "500k_384d_titan_mm_v1": {
    "mappings": {
      "dynamic": "strict",
      "properties": {
        "AgeSynonymFilter": { "type": "text", "analyzer": "age-synonym-analyzer" },
        "AllDistrib": { "type": "boolean", "doc_values": false },
        "AssetStatus": { "type": "keyword", "doc_values": false },
        "AssetType": { "type": "keyword" },
        "BValue": { "type": "short", "doc_values": false },
        "BlackAndWhite": { "type": "boolean" },
        "Caption": { "type": "text", "copy_to": ["MiscSynonymFilter", "EthnicitySynonymFilter", "AgeSynonymFilter"] },
        "CaptionDe": { "type": "text", "index": false },
        "CaptionFr": { "type": "text", "index": false },
        "CaptionIt": { "type": "text", "index": false },
        "CompanyName": { "type": "text" },
        "ContainsNoOfPeople": { "type": "short", "doc_values": false },
        "ContainsProperty": { "type": "boolean", "doc_values": false },
        "ContributorRef": { "type": "text" },
        "ContributorType": { "type": "keyword", "doc_values": false },
        "Dataco": { "type": "boolean", "doc_values": false },
        "DateTaken": { "type": "date", "format": "rfc3339_lenient" },
        "Description": { "type": "text", "copy_to": ["MiscSynonymFilter", "EthnicitySynonymFilter", "AgeSynonymFilter"] },
        "DiscountLibraries": { "type": "keyword" },
        "DistribTerritories": { "type": "keyword", "doc_values": false },
        "EssentialWords": { "type": "text", "copy_to": ["MiscSynonymFilter", "EthnicitySynonymFilter", "AgeSynonymFilter"] },
        "EthnicitySynonymFilter": { "type": "text", "analyzer": "ethnicity-synonym-analyzer" },
        "ExclusiveToAlamy": { "type": "boolean", "doc_values": false },
        "ExtraWords": { "type": "text", "copy_to": ["MiscSynonymFilter", "EthnicitySynonymFilter", "AgeSynonymFilter"] },
        "GValue": { "type": "short", "doc_values": false },
        "GeoRestrictions": { "type": "keyword", "doc_values": false },
        "Id": { "type": "keyword", "doc_values": false },
        "ImageBoost": { "type": "short", "doc_values": false },
        "IsACutOut": { "type": "boolean" },
        "IsArchive": { "type": "boolean", "doc_values": false },
        "IsEditorialOnly": { "type": "boolean", "doc_values": false },
        "IsMobile": { "type": "boolean", "doc_values": false },
        "IsNews": { "type": "boolean", "doc_values": false },
        "IsNovelUseRegistered": { "type": "boolean", "doc_values": false },
        "IsPubliclyAvailable": { "type": "boolean", "doc_values": false },
        "LastModified": { "type": "date", "format": "rfc3339_lenient" },
        "Libraries": { "type": "keyword" },
        "LibraryLicenses": { "type": "keyword", "doc_values": false },
        "LicenseType": { "type": "keyword", "doc_values": false },
        "Location": { "type": "text" },
        "MainWords": { "type": "text", "copy_to": ["MiscSynonymFilter", "EthnicitySynonymFilter", "AgeSynonymFilter"] },
        "MiscSynonymFilter": { "type": "text", "analyzer": "prohibited-synonym-analyzer" },
        "ModelRelease": { "type": "boolean", "doc_values": false },
        "Orientation": { "type": "keyword", "doc_values": false },
        "PixelsX": { "type": "integer", "doc_values": false },
        "PixelsY": { "type": "integer", "doc_values": false },
        "PrimaryCategory": { "type": "keyword", "doc_values": false },
        "PropertyRelease": { "type": "boolean", "doc_values": false },
        "PseudoId": { "type": "keyword", "doc_values": false },
        "Pseudonym": { "type": "text" },
        "PseudonymRank": { "type": "short" },
        "RValue": { "type": "short", "doc_values": false },
        "Ref": { "type": "text" },
        "SearchCollectionId": { "type": "keyword", "doc_values": false },
        "Searchmap": { "type": "keyword" },
        "SecondaryCategory": { "type": "keyword", "doc_values": false },
        "Sequence": { "type": "long" },
        "Silo": { "type": "keyword", "doc_values": false },
        "SizeInBytesUncompressed": { "type": "integer", "doc_values": false },
        "SizeInMb": { "type": "float", "doc_values": false },
        "UnifiedDateTaken": { "type": "date", "format": "rfc3339_lenient" },
        "UploadDate": { "type": "date", "format": "rfc3339_lenient" },
        "UserId": { "type": "keyword" },
        "Vector": {
          "type": "knn_vector",
          "dimension": 384,
          "method": {
            "engine": "faiss",
            "space_type": "l2",
            "name": "hnsw",
            "parameters": {
              "ef_search": 256,
              "ef_construction": 256,
              "m": 32
            }
          }
        }
      }
    }
  }
}

```

---

Would you like me to help you refine the search queries or explain any specific part of the mapping?
