

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
