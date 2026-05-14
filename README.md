# Elasticsearch Comprehensive Cheatsheet

> **Comprehensive Elasticsearch cheatsheet covering everything from basic concepts, CRUD operations, and query DSL to aggregations, mappings, performance tuning, security, and monitoring. Includes practical examples for indexing, searching, filtering, bulk operations, reindexing, and cluster health management. Essential commands, cURL examples, and optimization tips for developers and DBAs.**

## Table of Contents

1. [Basic Concepts](#1-basic-concepts)
2. [Installation & Setup](#2-installation--setup)
3. [Index Management](#3-index-management)
4. [Document CRUD Operations](#4-document-crud-operations)
5. [Mapping & Data Types](#5-mapping--data-types)
6. [Query DSL Basics](#6-query-dsl-basics)
7. [Full-Text Search Queries](#7-full-text-search-queries)
8. [Term-Level Queries](#8-term-level-queries)
9. [Compound Queries](#9-compound-queries)
10. [Aggregations](#10-aggregations)
11. [Filters vs Queries](#11-filters-vs-queries)
12. [Sorting & Pagination](#12-sorting--pagination)
13. [Analyzers & Tokenizers](#13-analyzers--tokenizers)
14. [Index Settings](#14-index-settings)
15. [Reindexing & Aliases](#15-reindexing--aliases)
16. [Performance Optimization](#16-performance-optimization)
17. [Monitoring & Health](#17-monitoring--health)
18. [Security & Authentication](#18-security--authentication)
19. [Common REST API Endpoints](#19-common-rest-api-endpoints)
20. [Useful cURL Examples](#20-useful-curl-examples)

---

## 1. Basic Concepts

| Concept | Equivalent in SQL | Description |
|---------|------------------|-------------|
| Index | Database | Collection of documents |
| Type | Table | (Deprecated in 7.x, removed in 8.x) |
| Document | Row | JSON object stored |
| Field | Column | Key-value pair in document |
| Mapping | Schema | Defines field types |
| Shard | Partition | Horizontal data split |
| Replica | Backup | Copy of shard for HA |

**Cluster States:**
- **Green** - All shards allocated & working
- **Yellow** - Replicas not allocated (still operational)
- **Red** - Primary shards missing (data loss risk)

---

## 2. Installation & Setup

### Docker
```bash
# Single node
docker run -d --name elasticsearch \
  -p 9200:9200 -p 9300:9300 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  elasticsearch:8.11.0

# With security (8.x+ default)
docker run -d --name elasticsearch \
  -p 9200:9200 -p 9300:9300 \
  -e "ELASTIC_PASSWORD=yourpassword" \
  -e "xpack.security.enabled=true" \
  elasticsearch:8.11.0
```

### Important Settings (elasticsearch.yml)
```yaml
cluster.name: my-cluster
node.name: node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["host1", "host2"]
cluster.initial_master_nodes: ["node-1"]
xpack.security.enabled: true
```

### Verify Installation
```bash
curl -X GET "localhost:9200/"
curl -X GET "localhost:9200/_cluster/health?pretty"
```

---

## 3. Index Management

### Create Index
```json
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2
  },
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "price": { "type": "float" },
      "in_stock": { "type": "boolean" }
    }
  }
}
```

### List Indices
```bash
GET /_cat/indices?v
GET /_aliases?pretty
```

### Delete Index
```bash
DELETE /products
DELETE /products,orders    # Multiple
DELETE /_all               # Delete all (dangerous!)
DELETE /*                  # Delete all (wildcard)
```

### Index Settings (Update)
```json
PUT /products/_settings
{
  "index": {
    "number_of_replicas": 3,
    "refresh_interval": "30s"
  }
}
```

### Open/Close Index
```bash
POST /products/_close
POST /products/_open
```

### Index Stats
```bash
GET /products/_stats
GET /_nodes/stats
GET /_cluster/stats
```

---

## 4. Document CRUD Operations

### Create (Index) Document
```json
# Auto-generated ID
POST /products/_doc
{
  "name": "Laptop",
  "price": 999.99,
  "in_stock": true
}

# Custom ID
PUT /products/_doc/1
{
  "name": "Mouse",
  "price": 25.50,
  "in_stock": true
}

# Create - fails if document exists
PUT /products/_create/1
{ "name": "Keyboard" }
```

### Read Document
```bash
# Get by ID
GET /products/_doc/1
GET /products/_source/1  # Returns only source

# Check existence
HEAD /products/_doc/1
```

### Update Document
```json
# Partial update
POST /products/_update/1
{
  "doc": {
    "price": 29.99
  }
}

# Update with script (increment)
POST /products/_update/1
{
  "script": {
    "source": "ctx._source.price += params.increment",
    "params": { "increment": 5 }
  }
}

# Upsert (update or insert)
POST /products/_update/1
{
  "script": { "source": "ctx._source.price++" },
  "upsert": { "name": "New Item", "price": 10 }
}
```

### Delete Document
```bash
DELETE /products/_doc/1

# Delete by query
POST /products/_delete_by_query
{
  "query": { "term": { "in_stock": false } }
}
```

### Bulk Operations
```json
POST /_bulk
{ "index": { "_index": "products", "_id": "1" } }
{ "name": "Laptop", "price": 1000 }
{ "update": { "_index": "products", "_id": "2" } }
{ "doc": { "price": 500 } }
{ "delete": { "_index": "products", "_id": "3" } }
```

---

## 5. Mapping & Data Types

### Common Data Types

| Type | Description | Example |
|------|-------------|---------|
| `text` | Analyzed full-text | "The quick brown fox" |
| `keyword` | Exact value (not analyzed) | "USA", "ACTIVE" |
| `integer`, `long` | Whole numbers | 42 |
| `float`, `double` | Floating point | 3.14 |
| `boolean` | True/false | true |
| `date` | ISO 8601 or epoch | "2024-01-15" |
| `object` | Nested JSON | { "city": "NYC" } |
| `nested` | Array of objects (independent) | [ { "name": "a" }, { "name": "b" } ] |
| `geo_point` | Geo coordinates | { "lat": 40.71, "lon": -74.01 } |
| `ip` | IPv4/IPv6 | "192.168.1.1" |

### Create Mapping
```json
PUT /users
{
  "mappings": {
    "properties": {
      "email": { "type": "keyword" },
      "bio": {
        "type": "text",
        "analyzer": "english"
      },
      "age": { "type": "integer" },
      "created_at": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss"
      },
      "address": {
        "type": "object",
        "properties": {
          "city": { "type": "text" },
          "zipcode": { "type": "keyword" }
        }
      },
      "tags": { "type": "keyword" },
      "location": { "type": "geo_point" }
    }
  }
}
```

### View Mapping
```bash
GET /users/_mapping
GET /users/_mapping/field/email
```

### Add New Field to Mapping
```json
PUT /users/_mapping
{
  "properties": {
    "phone": { "type": "keyword" }
  }
}
```

### Dynamic Mapping Control
```json
PUT /strict_index
{
  "mappings": {
    "dynamic": "strict",  # strict, true, false
    "properties": {
      "name": { "type": "text" }
    }
  }
}
```

---

## 6. Query DSL Basics

### Match All
```json
GET /products/_search
{
  "query": { "match_all": {} }
}
```

### Pagination
```json
GET /products/_search
{
  "from": 10,
  "size": 20,
  "query": { "match_all": {} }
}
```

### Source Filtering
```json
GET /products/_search
{
  "_source": ["name", "price"],
  "query": { "match_all": {} }
}

# Exclude fields
{
  "_source": { "excludes": ["description"] }
}
```

### Search Response Fields
```json
{
  "took": 5,                    # Milliseconds
  "timed_out": false,
  "_shards": { "total": 1, "successful": 1, "skipped": 0, "failed": 0 },
  "hits": {
    "total": { "value": 100, "relation": "eq" },
    "max_score": 1.0,
    "hits": [ ... ]
  }
}
```

---

## 7. Full-Text Search Queries

### Match Query (Standard)
```json
GET /products/_search
{
  "query": {
    "match": {
      "description": "wireless bluetooth mouse"
    }
  }
}

# With operator
{
  "match": {
    "description": {
      "query": "fast cheap laptop",
      "operator": "and"  # All terms must match
    }
  }
}
```

### Match Phrase
```json
{
  "match_phrase": {
    "description": "high performance laptop"
  }
}
```

### Multi-Match
```json
{
  "multi_match": {
    "query": "wireless mouse",
    "fields": ["name", "description", "tags^3"]  # ^3 boosts weight
  }
}
```

### Match Phrase Prefix (Autocomplete)
```json
{
  "match_phrase_prefix": {
    "name": "wireless mou"
  }
}
```

### Query String
```json
{
  "query_string": {
    "query": "(laptop OR mouse) AND price:>500",
    "fields": ["name", "description"]
  }
}
```

### Simple Query String
```json
{
  "simple_query_string": {
    "query": "laptop +mouse -keyboard",
    "fields": ["name"],
    "default_operator": "and"
  }
}
```

---

## 8. Term-Level Queries

### Term Query (Exact match)
```json
GET /products/_search
{
  "query": {
    "term": {
      "status.keyword": "ACTIVE"  # Use .keyword for exact match
    }
  }
}
```

### Terms Query (Multiple values)
```json
{
  "terms": {
    "category": ["electronics", "computers"]
  }
}
```

### Range Query
```json
{
  "range": {
    "price": {
      "gte": 100,
      "lte": 500,
      "boost": 2.0
    }
  }
}
```

### Exists Query
```json
{
  "exists": {
    "field": "description"
  }
}
```

### Prefix Query
```json
{
  "prefix": {
    "brand.keyword": "Sam"
  }
}
```

### Wildcard Query
```json
{
  "wildcard": {
    "name.keyword": "lapt*p"
  }
}
```

### Regexp Query
```json
{
  "regexp": {
    "code": "A[0-9]{3}B"
  }
}
```

### IDs Query
```json
{
  "ids": {
    "values": ["1", "2", "3"]
  }
}
```

---

## 9. Compound Queries

### Bool Query
```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [                    # AND - contributes to score
        { "match": { "name": "laptop" } }
      ],
      "should": [                  # OR - increases score
        { "term": { "brand": "Dell" } },
        { "term": { "brand": "HP" } }
      ],
      "must_not": [                # NOT - excludes documents
        { "term": { "status": "discontinued" } }
      ],
      "filter": [                  # AND - no scoring, cacheable
        { "range": { "price": { "lte": 1000 } } }
      ],
      "minimum_should_match": 1    # At least 1 should clause
    }
  }
}
```

### Boosting Query
```json
{
  "boosting": {
    "positive": { "match": { "name": "laptop" } },
    "negative": { "match": { "name": "broken" } },
    "negative_boost": 0.2
  }
}
```

### Constant Score (Wrap filter)
```json
{
  "constant_score": {
    "filter": { "term": { "active": true } },
    "boost": 1.2
  }
}
```

### Dis Max (Best fields)
```json
{
  "dis_max": {
    "queries": [
      { "match": { "title": "quick brown" } },
      { "match": { "body": "quick brown" } }
    ],
    "tie_breaker": 0.3
  }
}
```

---

## 10. Aggregations

### Bucket Aggregations

#### Terms Aggregation
```json
GET /sales/_search
{
  "size": 0,  # Don't return hits
  "aggs": {
    "popular_products": {
      "terms": {
        "field": "product_id.keyword",
        "size": 10,
        "order": { "_count": "desc" }
      }
    }
  }
}
```

#### Range Aggregation
```json
{
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 50 },
          { "from": 50, "to": 100 },
          { "from": 100 }
        ]
      }
    }
  }
}
```

#### Date Histogram
```json
{
  "aggs": {
    "sales_over_time": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "month",
        "format": "yyyy-MM"
      }
    }
  }
}
```

#### Missing Values
```json
{
  "aggs": {
    "without_category": {
      "missing": { "field": "category" }
    }
  }
}
```

### Metric Aggregations

```json
GET /sales/_search
{
  "size": 0,
  "aggs": {
    "avg_price": { "avg": { "field": "price" } },
    "total_revenue": { "sum": { "field": "price" } },
    "min_price": { "min": { "field": "price" } },
    "max_price": { "max": { "field": "price" } },
    "price_stats": { "stats": { "field": "price" } },
    "cardinality": { "cardinality": { "field": "user_id.keyword" } }
  }
}
```

### Nested Aggregations
```json
{
  "aggs": {
    "by_country": {
      "terms": { "field": "country.keyword" },
      "aggs": {
        "avg_spending": { "avg": { "field": "amount" } },
        "top_products": {
          "terms": { "field": "product_id.keyword", "size": 5 }
        }
      }
    }
  }
}
```

### Pipeline Aggregations
```json
{
  "aggs": {
    "sales_per_month": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "month"
      },
      "aggs": {
        "total": { "sum": { "field": "amount" } }
      }
    },
    "moving_avg": {
      "moving_avg": {
        "buckets_path": "sales_per_month>total",
        "window": 30
      }
    }
  }
}
```

---

## 11. Filters vs Queries

| Feature | Query | Filter |
|---------|-------|--------|
| Scoring | Yes (calculates relevance) | No |
| Caching | Not cached | Automatically cached |
| Performance | Slower | Faster |
| Use case | Full-text search | Exact matches, ranges |

### Filter Context Example
```json
{
  "query": {
    "bool": {
      "must": { "match": { "title": "search" } },  # Query
      "filter": {                                  # Filter
        "range": { "price": { "lte": 100 } }
      }
    }
  }
}
```

### Pure Filter (Constant Score)
```json
{
  "query": {
    "constant_score": {
      "filter": { "term": { "status": "active" } }
    }
  }
}
```

---

## 12. Sorting & Pagination

### Basic Sort
```json
GET /products/_search
{
  "sort": [
    { "price": { "order": "asc" } },
    { "popularity": { "order": "desc" } }
  ],
  "query": { "match_all": {} }
}
```

### Sort with Missing Values
```json
{
  "sort": [
    {
      "price": {
        "order": "asc",
        "missing": "_last"  # or "_first"
      }
    }
  ]
}
```

### Multi-level Sorting
```json
{
  "sort": [
    { "category.keyword": "asc" },
    { "price": "desc" }
  ]
}
```

### Search After (Deep Pagination)
```json
# First query
GET /products/_search
{
  "size": 10,
  "sort": [{ "price": "asc" }, { "_id": "asc" }],
  "query": { "match_all": {} }
}

# Subsequent query using search_after
GET /products/_search
{
  "size": 10,
  "sort": [{ "price": "asc" }, { "_id": "asc" }],
  "search_after": [99.99, "product_123"],
  "query": { "match_all": {} }
}
```

### Scroll API (Large Datasets)
```json
# Initialize scroll
GET /products/_search?scroll=2m
{
  "size": 1000,
  "query": { "match_all": {} }
}

# Continue scrolling
GET /_search/scroll
{
  "scroll": "2m",
  "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gB..."
}

# Clear scroll
DELETE /_search/scroll
{
  "scroll_id": "DXF1ZXJ5..."
}
```

### Pit (Point in Time - 7.x+)
```json
# Create PIT
POST /products/_pit?keep_alive=5m

# Search with PIT
GET /_search
{
  "pit": { "id": "pit_id_here", "keep_alive": "5m" },
  "sort": [{ "price": "asc" }],
  "search_after": [99.99]
}
```

---

## 13. Analyzers & Tokenizers

### Built-in Analyzers
```json
# Standard (default)
POST /_analyze
{
  "analyzer": "standard",
  "text": "The quick brown foxes!"
}

# Simple (split on non-letters, lowercase)
{ "analyzer": "simple", "text": "The 2 QUICK Brown-Foxes" }

# Whitespace
{ "analyzer": "whitespace", "text": "The quick brown fox" }

# Keyword (single token, no change)
{ "analyzer": "keyword", "text": "Hello World!" }

# Stop (removes stop words)
{ "analyzer": "stop", "text": "The quick brown fox" }

# Language-specific (English)
{ "analyzer": "english", "text": "running foxes" }  # "run", "fox"
```

### Custom Analyzer
```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "custom_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "stop", "asciifolding"],
          "char_filter": ["html_strip"]
        }
      }
    }
  }
}
```

### Common Token Filters
- `lowercase` - Convert to lowercase
- `stop` - Remove stop words
- `stemmer` - Reduce to word stems
- `synonym` - Apply synonyms
- `ngram` - Create n-grams
- `edge_ngram` - Edge n-grams (autocomplete)

### Test Analyzer
```bash
POST /my_index/_analyze
{
  "field": "description",
  "text": "Running in the park"
}
```

---

## 14. Index Settings

### Important Settings

| Setting | Default | Description |
|---------|---------|-------------|
| `number_of_shards` | 1 | Primary shards (set at creation) |
| `number_of_replicas` | 1 | Copy count per shard |
| `refresh_interval` | 1s | How often index is refreshed |
| `index.max_result_window` | 10000 | Max from+size |
| `index.mapping.total_fields.limit` | 1000 | Max fields per index |
| `index.codec` | default | best_compression for LZ4 |

### Update Settings
```json
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "30s",
    "number_of_replicas": 2,
    "max_result_window": 50000
  }
}
```

### Read-Only Index
```json
PUT /my_index/_settings
{
  "index.blocks.write": true
}
```

### Slow Log Settings
```json
PUT /my_index/_settings
{
  "index.search.slowlog.threshold.query.warn": "10s",
  "index.search.slowlog.threshold.query.info": "5s",
  "index.indexing.slowlog.threshold.index.warn": "10s"
}
```

---

## 15. Reindexing & Aliases

### Reindex (Copy data)
```json
POST /_reindex
{
  "source": { "index": "old_products" },
  "dest": { "index": "new_products" }
}

# With query filter
POST /_reindex
{
  "source": {
    "index": "logs",
    "query": { "range": { "@timestamp": { "gte": "2024-01-01" } } }
  },
  "dest": { "index": "archive_logs" }
}

# Reindex with transformation
POST /_reindex
{
  "source": { "index": "products" },
  "dest": { "index": "new_products" },
  "script": {
    "source": "ctx._source.price = ctx._source.price * 1.1"
  }
}
```

### Update by Query
```json
POST /products/_update_by_query
{
  "script": {
    "source": "ctx._source.in_stock = false"
  },
  "query": {
    "range": { "stock_count": { "lte": 0 } }
  }
}
```

### Index Aliases
```json
# Create alias
POST /_aliases
{
  "actions": [
    { "add": { "index": "products_v1", "alias": "products" } }
  ]
}

# Swap alias (zero-downtime reindex)
POST /_aliases
{
  "actions": [
    { "remove": { "index": "products_v1", "alias": "products" } },
    { "add": { "index": "products_v2", "alias": "products" } }
  ]
}

# Filter alias
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "logs",
        "alias": "error_logs",
        "filter": { "term": { "level": "ERROR" } }
      }
    }
  ]
}
```

---

## 16. Performance Optimization

### Indexing Performance
```json
# Bulk instead of individual requests
# Increase refresh interval during large imports
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "-1",  # Disable during bulk load
    "number_of_replicas": 0
  }
}

# After import, revert
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "30s",
    "number_of_replicas": 1
  }
}
```

### Search Performance Tips
1. Use **filters** instead of queries when scoring not needed
2. Avoid wildcard queries on large fields
3. Set `_source` to include only needed fields
4. Use `keyword` fields for exact matches
5. Limit shard size (20-50GB per shard)
6. Prefer `match_phrase` over `query_string`
7. Use `search_after` instead of deep `from/size`

### Force Merge Segments
```bash
POST /my_index/_forcemerge?max_num_segments=1

# Only for read-only indices
POST /logs_2023/_forcemerge?only_expunge_deletes=true
```

### Node Configuration
```yaml
# JVM heap (set to 50% of RAM, max 32GB)
ES_JAVA_OPTS="-Xms8g -Xmx8g"

# Disable swapping
bootstrap.memory_lock: true

# File system cache (let ES manage)
```

---

## 17. Monitoring & Health

### Cluster Health
```bash
GET /_cluster/health?pretty
GET /_cluster/health?level=indices
GET /_cluster/health?level=shards
GET /_cluster/health/wait_for_status=yellow&timeout=30s
```

### Node Statistics
```bash
GET /_nodes/stats
GET /_nodes/stats/indices,os,process,jvm
GET /_nodes/hot_threads
```

### Index Statistics
```bash
GET /_cat/indices?v&s=store.size:desc
GET /_cat/shards/my_index?v
GET /_cat/segments/my_index?v
```

### Pending Tasks
```bash
GET /_cluster/pending_tasks
```

### Task Management
```bash
# List tasks
GET /_tasks?detailed=true&actions=*reindex

# Cancel task
POST /_tasks/task_id:12345/_cancel
```

### Common Monitoring Metrics
```bash
# Disk usage
GET /_cat/allocation?v

# Node hot threads
GET /_nodes/hot_threads?interval=500ms

# JVM stats
GET /_nodes/stats/jvm?pretty

# Query/Index latency
GET /_nodes/stats/search?pretty
GET /_nodes/stats/indexing?pretty
```

---

## 18. Security & Authentication

### Built-in Users (8.x)
```bash
# Reset password
elasticsearch-reset-password -u elastic

# Create new user
POST /_security/user/johndoe
{
  "password": "securepassword",
  "roles": ["superuser"],
  "full_name": "John Doe"
}
```

### Roles & Privileges
```json
# Create role
POST /_security/role/readonly_user
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read", "view_index_metadata"],
      "field_security": { "grant": ["timestamp", "message"], "except": ["password"] }
    }
  ]
}
```

### API Key Authentication
```bash
# Create API key
POST /_security/api_key
{
  "name": "my-api-key",
  "role_descriptors": {
    "read_only": {
      "cluster": ["monitor"],
      "indices": [{ "names": ["*"], "privileges": ["read"] }]
    }
  }
}
```

### TLS/SSL Configuration
```yaml
# elasticsearch.yml
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
xpack.security.http.ssl.enabled: true
```

---

## 19. Common REST API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/_cat/indices` | GET | List indices |
| `/_cluster/health` | GET | Cluster health |
| `/_cluster/state` | GET | Cluster state |
| `/_cluster/settings` | GET/PUT | Cluster settings |
| `/_nodes/stats` | GET | Node stats |
| `/_reindex` | POST | Reindex operation |
| `/_alias` | POST | Manage aliases |
| `/_analyze` | POST | Test analyzers |
| `/_validate/query` | POST | Validate queries |
| `/_count` | POST | Count documents |
| `/_search` | POST | Search across indices |
| `/_bulk` | POST | Bulk operations |
| `/_mget` | POST | Multi-get |
| `/_refresh` | POST | Refresh indices |
| `/_flush` | POST | Flush to disk |
| `/_snapshot` | PUT/GET | Backup management |
| `/_enrich` | PUT | Enrich policies |
| `/_ilm/policy` | PUT | Index Lifecycle Management |
| `/_rollover` | POST | Rollover index |

---

## 20. Useful cURL Examples

### Basic Operations
```bash
# Health check
curl -X GET "localhost:9200/_cluster/health?pretty"

# Create index
curl -X PUT "localhost:9200/products" -H 'Content-Type: application/json' -d'
{
  "settings": { "number_of_shards": 3 },
  "mappings": { "properties": { "name": { "type": "text" } } }
}'

# Index document
curl -X POST "localhost:9200/products/_doc" -H 'Content-Type: application/json' -d'
{"name": "Laptop", "price": 999}'

# Search
curl -X GET "localhost:9200/products/_search?q=name:laptop&pretty"

# Query DSL
curl -X POST "localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
{
  "query": { "match": { "name": "laptop" } }
}'

# Delete by query
curl -X POST "localhost:9200/products/_delete_by_query" -H 'Content-Type: application/json' -d'
{
  "query": { "term": { "price": 0 } }
}'
```

### Bulk Import
```bash
curl -X POST "localhost:9200/_bulk" -H 'Content-Type: application/json' --data-binary "@bulk_data.json"
```

### Export Data
```bash
# Export to file
curl -X POST "localhost:9200/products/_search?size=10000" -H 'Content-Type: application/json' -d'
{
  "query": { "match_all": {} }
}' > products_export.json
```

### Backup with Snapshot
```bash
# Register repository
curl -X PUT "localhost:9200/_snapshot/my_backup" -H 'Content-Type: application/json' -d'
{
  "type": "fs",
  "settings": { "location": "/mount/backups/my_backup" }
}'

# Create snapshot
curl -X PUT "localhost:9200/_snapshot/my_backup/snapshot_1?wait_for_completion=true"

# Restore snapshot
curl -X POST "localhost:9200/_snapshot/my_backup/snapshot_1/_restore" -H 'Content-Type: application/json' -d'
{
  "indices": "products",
  "ignore_unavailable": true,
  "include_global_state": false
}'
```

### Node Maintenance
```bash
# Exclude node from allocation
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "transient": {
    "cluster.routing.allocation.exclude._name": "node-1"
  }
}'

# Enable shard allocation
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "transient": {
    "cluster.routing.allocation.enable": "all"
  }
}'
```

### Clean Up
```bash
# Delete old indices by date
curl -X DELETE "localhost:9200/logs-2023*"

# Clear cache
curl -X POST "localhost:9200/_cache/clear"

# Force merge
curl -X POST "localhost:9200/products/_forcemerge?max_num_segments=1"
```

---

## Quick Reference Card

| Operation | Command |
|-----------|---------|
| View health | `GET /_cluster/health` |
| List indices | `GET /_cat/indices` |
| Search all | `GET /_search` |
| Count docs | `GET /index/_count` |
| Delete index | `DELETE /index` |
| Refresh index | `POST /index/_refresh` |
| Clear cache | `POST /_cache/clear` |
| Node stats | `GET /_nodes/stats` |

**Useful Query Patterns:**
```json
// Full-text search
{ "match": { "field": "text" } }

// Exact match
{ "term": { "field.keyword": "value" } }

// Range
{ "range": { "price": { "gte": 10 } } }

// Bool combination
{ "bool": { "must": [], "filter": [], "must_not": [] } }

// Aggregation
{ "aggs": { "name": { "terms": { "field": "field.keyword" } } } }
```

---

**Pro Tips:**
1. Always use `?pretty` for readable JSON output
2. Set `size=0` when only aggregations are needed
3. Use `_source` filtering to reduce response size
4. Prefer `keyword` over `text` for exact matches
5. Monitor shard sizes - keep between 10-50GB
6. Use ILM (Index Lifecycle Management) for time-series data
7. Enable slow logs in production
8. Test queries with `?explain` for debugging

---

This cheatsheet covers the most common Elasticsearch operations. For detailed documentation, visit: [https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
