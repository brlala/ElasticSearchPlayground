### Inspecting the cluster
1. GET /_cluster/health - cluster_name, status, node counts, relocating status, etc
2. GET /_cat/nodes?v - ipaddress, performance
3. GET /_cat/indices?v - look at indices, primary shards, document count, size etc.

### Index management
1. PUT /products - add a `products` index, default to 1 primary, 1 replica
```json
//body
{
  "settings": {
    "number_of_shards": 2, 
    "number_of_replicas": 2
  }
}
```
2. POST /products/_doc - creating a document 
```json
//body
{
  "name": "Coffee Maker",
  "price": 64,
  "in_stock": 10
}

//result
{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "f-E5D3sBKMY-Mln_dJli",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 3,  //this count include those of a replica
    "successful" : 3,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}
```
3. PUT /products/_doc/100 - creating a document specifying id, same with replacing
```json
//body
{
  "name": "Toaster",
  "price": 49,
  "in_stock": 4
}

//results
{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "100",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 3,
    "successful" : 3,
    "failed" : 0
  },
  "_seq_no" : 1,
  "_primary_term" : 1
}
```
3. GET /products/_doc/100 - retrieve from `products` index for document id `100`
```json
//results
{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "100",
  "_version" : 1,
  "_seq_no" : 1,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "name" : "Toaster",
    "price" : 49,
    "in_stock" : 4
  }
}
```
4. POST /products/_update/100 - updating documents field
```json
//body
{
  "doc": {
    "in_stock": 3
  }
}

//results
{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "100",
  "_version" : 2,
  "result" : "updated",  // might have value of "noop" if there is no change
  "_shards" : {
    "total" : 3,
    "successful" : 3,
    "failed" : 0
  },
  "_seq_no" : 2,
  "_primary_term" : 1
}
```
5. POST /products/_update/100 - updating documents field
```json
//body
{
  "doc": {
    "in_stock": 3
  }
}

//results
{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "100",
  "_version" : 2,
  "result" : "updated",  // might have value of "noop" if there is no change
  "_shards" : {
    "total" : 3,
    "successful" : 3,
    "failed" : 0
  },
  "_seq_no" : 2,
  "_primary_term" : 1
}
```
6. POST /products/_update/100 - Scripted update can be done by changing the body, if can support if conditions and more, do note that `noop` will NOT show even if the document is unchanged, unless we specify a `noop` context. see sample
````json
//body(deduct by 1)
{
  "script": {
    "source": "ctx._source.in_stock--" //access source document, in_stock field and subtract one from it
  }
}

//body(deduct by 1)
{
  "script": {
    "source": "ctx._source.in_stock = 10"
  }
}

//body(using params)
{
  "script": {
    "source": "ctx._source.in_stock -= params.quantity",
    "params": {
      "quantity": 4
    }
  }
}

//body(noop)
{
  "script": {
    "source": """
      if (ctx._source.in_stock == 0) {
        ctx.op = 'noop';
      }
      ctx._source.in_stock--;
    """
  }
}
            
//body(noop)
{
  "script": {
    "source": """
      if (ctx._source.in_stock > 0) {
        ctx._source.in_stock--;
      }
    """
  }
}

//results
{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "100",
  "_version" : 3,
  "result" : "updated",
  "_shards" : {
    "total" : 3,
    "successful" : 3,
    "failed" : 0
  },
  "_seq_no" : 3,
  "_primary_term" : 1
}
````
7. POST /products/_update/100 - UPSERT, will insert the document if doesn't exist else minus fields
```json
//body
{
  "script": {
    "source": "ctx._source.in_stock--"
  },
  "upsert": {
    "name": "Blender",
    "price": 399,
    "in_stock": 6
  }
}
```
8. DELETE /products/_doc/100
```json
//results
{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "101",
  "_version" : 4,
  "result" : "deleted",
  "_shards" : {
    "total" : 3,
    "successful" : 3,
    "failed" : 0
  },
  "_seq_no" : 15,
  "_primary_term" : 1
}

//results 404 error
{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "100",
  "_version" : 16,
  "result" : "not_found",
  "_shards" : {
    "total" : 3,
    "successful" : 3,
    "failed" : 0
  },
  "_seq_no" : 23,
  "_primary_term" : 1
}
```
9. POST /products/_update/101?if_primary_term=1&if_seq_no=24 - Concurrency Control, must match the pimary term and seq_number when updating else it will return a 409 conflict
```json
//body
{
  "script": {
    "source": "ctx._source.in_stock--"
  }
}

//result(success)
{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "101",
  "_version" : 3,
  "result" : "updated",
  "_shards" : {
    "total" : 3,
    "successful" : 3,
    "failed" : 0
  },
  "_seq_no" : 26,
  "_primary_term" : 1
}

//result(fail) 409 error
{
  "error" : {
    "root_cause" : [
      {
        "type" : "version_conflict_engine_exception",
        "reason" : "[101]: version conflict, required seqNo [24], primary term [1]. current document has seqNo [25] and primary term [1]",
        "index_uuid" : "-koecmIMQFqbp27DOmQnMg",
        "shard" : "0",
        "index" : "products"
      }
    ],
    "type" : "version_conflict_engine_exception",
    "reason" : "[101]: version conflict, required seqNo [24], primary term [1]. current document has seqNo [25] and primary term [1]",
    "index_uuid" : "-koecmIMQFqbp27DOmQnMg",
    "shard" : "0",
    "index" : "products"
  },
  "status" : 409
}
```
10. POST /products/_update_by_query - Updating by query, using where clause'
    1. When a search query is send to the node, the node takes a snapshot of the index
    2. search query is sent to each shards.
    3. When a search query is matched, a bulk query is sent to the shard.
    4. Each pair of search+bulk query is sent sequentially, one at a time
    5. if there are failures, the whole query is aborted(not rolledback, if document is updated it stays updated)
```json
// body
{
  "conflicts": "proceed", // if found version conflict, will count and continue instead of abort
  "script": {
    "source": "ctx._source.in_stock--"
  },
  "query": {
    "match_all": {}
  }
}

// results
{
  "took" : 2572,
  "timed_out" : false,
  "total" : 2,
  "updated" : 2,
  "deleted" : 0,
  "batches" : 1,  //how many batches were used to retrieve the documents,uses Scroll API internally
  "version_conflicts" : 0,  // number of conflicts updated item != snapshot created
  "noops" : 0,
  "retries" : {
    "bulk" : 0,
    "search" : 0
  },
  "throttled_millis" : 0,
  "requests_per_second" : -1.0,
  "throttled_until_millis" : 0,
  "failures" : [ ]
}
```

11. POST /products/_delete_by_query - deleting by query, using where clause'
```json
// body
{
  "conflicts": "proceed", // if found version conflict, will count and continue instead of abort
  "script": {
    "source": "ctx._source.in_stock--"
  },
  "query": {
    "match_all": {}
  }
}

// results
{
  "took" : 2572,
  "timed_out" : false,
  "total" : 2,
  "updated" : 2,
  "deleted" : 0,
  "batches" : 1,  //how many batches were used to retrieve the documents,uses Scroll API internally
  "version_conflicts" : 0,  // number of conflicts updated item != snapshot created
  "noops" : 0,
  "retries" : {
    "bulk" : 0,
    "search" : 0
  },
  "throttled_millis" : 0,
  "requests_per_second" : -1.0,
  "throttled_until_millis" : 0,
  "failures" : [ ]
}
```
11. POST /_bulk - Bulk API request, can take in routing parameters/concurrency control(primary terms/seq) etc. Use for sending multiple updates. Content-Type should be `ndjson`, multiple jsons  
`curl -H "Content-Type: application/x-ndjson" -XPOST http://localhost:9200/products/_bulk --data-binary "@products-bulk.json"`
```json
//body
POST /_bulk
{ "index": { "_index": "products", "_id": 200 } }  //index action will replace or create
{ "name" : "Expresso Machine", "price": 199, "in_stock": 5 }
{ "create": { "_index": "products", "_id": 201 } }
{ "name" : "Milk Frother", "price": 149, "in_stock": 14 }

//body(update to all products index)
POST /products/_bulk
{ "update": { "_id": 201 } }
{ "doc": { "price": 129 } }
{ "delete": { "_id": 200 } }
```
### Analyzer
1. Analyzer is used to add or change characters.
2. It can have zero or more `character filters` and they are applied in order, e.g. `html-strip` to strip html elements.
3. An analyzer can only contain one `tokenizer`, tokenize string, split sentence to words. It also records the character offset for each token in the original string.
4. An analyzer can contain one or more `token filter`. It receive the output from tokenizer as input. Can be used to apply filters e.g. `lowercase`
#### Types of analyzer
https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-analyzers.html
1. `standard`
   1. splits text at word boundaries and remove punctuation
   2. lowercase with lower case token filters
   3. remove stopwords(disabled by default)
2. `simple`
   1. splits into tokens when encountering anything else than letters
   2. lowercase tokenizer
3. `whitespace`
   1. does not lower case letter
   2. splits at space
4. `keyword`
   1. output as it is
5. `pattern`
   1. regex used to match token separators
   2. default pattern matches all non-word characters
   3. lowercase by default
```json
//body
POST /_analyze
{
  "text": "2 guys walk into a... bar :)",
  "analyzer": "standard"  // can be broken down into 3 components, other options "stemming_analyzer"
}

//body(specify own analyzer)
POST /_analyze
{
  "text": "2 guys walk into a... bar :)",
  "char_filter": [],
  "tokenizer": "standard",
  "filter": ["lowercase"]
}
//results
{
  "tokens" : [
    {
      "token" : "2",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "<NUM>",
      "position" : 0
    },
    {
      "token" : "guys",
      "start_offset" : 2,
      "end_offset" : 6,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "walk",
      "start_offset" : 7,
      "end_offset" : 11,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "into",
      "start_offset" : 12,
      "end_offset" : 16,
      "type" : "<ALPHANUM>",
      "position" : 3
    },
    {
      "token" : "a",
      "start_offset" : 17,
      "end_offset" : 18,
      "type" : "<ALPHANUM>",
      "position" : 4
    },
    {
      "token" : "bar",
      "start_offset" : 22,
      "end_offset" : 25,
      "type" : "<ALPHANUM>",
      "position" : 5
    }
  ]
}
```
5. Creating custom analyzer
```json
PUT /analyzer_test
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer": {
          "type": "custom",
          "char_filter": ["html_strip"],
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "stop",
            "asciifolding"
          ]
        }
      }
    }
  }
}
```
```json
PUT /analyzer_test
{
  "settings": {
    "analysis": {
      "filter": {
        "danish_stop": {  # filter named as danish stop
          "type": "stop",  # replace params
          "stopwords": "_danish_"
        }
      },
      "char_filter": {
        # Add character filters here
      },
      "tokenizer": {
        # Add tokenizers here
      },
      "analyzer": {
        "my_custom_analyzer": {
          "type": "custom",
          "char_filter": ["html_strip"],
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "danish_stop", # name of custom filter
            "asciifolding"
          ]
        }
      }
    }
  }
}
```
5. Adding/modifying analyzer to existing index, need to first close index
```json
//body close index
POST /analyzer_test/_close
```
```json
PUT /analyzer_test/_settings
{
  "analysis": {
    "analyzer": {
      "my_second_analyzer": {
        "type": "custom",
        "tokenizer": "standard",
        "char_filter": ["html_strip"],
        "filter": [
          "lowercase",
          "stop",
          "asciifolding"
        ]
      }
    }
  }
}
```
6. Each fields has it's own inverted index because different data uses different data structure for efficient searching e.g. BKD trees for numeric values, dates and geospatial data
####Data Type
1. Lucene does not support nested data hence the objects are flattened out. This will affect searching in fields.
   1. There are a few data types:
      1. `keyword` - used for filtering, aggregations and sorting, a `noop` `analyzer` is used.
          ```json
          {
              "text": "2 guys walk into a... bar :)",
              "analyzer": "keyword"
          }
          ```
      2. `nested` - when you want to search individual document in a document, enable us to query objects independently
      3. `text` - normal inverted index, used for full-text searches, searches that does not need to be exact
      4. `date` - support date wihout time("2015-03-27"), date with time("2015-04-15T13"07:41Z"), date with timezone("2015-04-15T13:07:41+01:00"), milliseconds since epoch(1436011284000), formatted according to ISO8601

#### Type coercion
1. Supplying a float will truncate to integer
2. Supplying "7.4" will create a text mapping
3. The value in the `_source` may not be the type used by elastic search for searching, it will retain as the field provided.
   1. Add a document with float value (dynamic mapping create a float value field)
   2. Add a document with string numbers (dynamic mapping will coerce for indexing but value remain as string)

#### Dynamic mapping
1. Nested arrays are flattened while indexing.
2. Arrays of strings are concatenated
3. All types in array must be same or coercable if dynamic mapping is enabled.

### Mapping Management
Documents are indexed as it is. Fields are optional even thou specified in mapping.
#### Mapping parameters
1. `format` - support multiple date formats like `epoch_second`
    ```json
    {
      "mappings": {
        "properties": {
          "rating": { "type": "float", "format": "dd/MM/yyyy"}
        }
      }
    }      
    ```
2. `coerce` - depends if coerce should be enabled or disabled(enabled by default)
    ```json
    // field level
    {
      "mappings": {
        "properties": {
          "rating": { "type": "float", "coerce": false}
        }
      }
    }
    
    // index level
    {
      "settings": {
        "index.mapping.coerce": false
      },
      "mappings": {
        "properties": {
          "rating": { "type": "float"}
        }
      }
    }
    ```
3. `doc_values` - additional data stored for aggregation/scripting/sorting, disable only if low disk space or increase throughput
    ```json
    // field level
    {
      "mappings": {
        "properties": {
          "rating": { "type": "float", "doc_values": false }
        }
      }
    }
3. `norms` - normalization factors used for relevance scoring, often we don't want to just filter but also rank them. It saves disk space but disable field for relevant scoring. Only use this if the field won't be used for relevance scoring
    ```json
    // field level
    {
      "mappings": {
        "properties": {
          "rating": { "type": "float", "norms": false }
        }
      }
    }
3. `index` - disable indexing for a field, but cannot be used in search query, save disk space and increase throughput, often used for time series data, can still be used for aggregation
    ```json
    // field level
    {
      "mappings": {
        "properties": {
          "rating": { "type": "float", "index": false }
        }
      }
    }
3. `null_value` - will default to this value when indexing and encounter `explicit` null value. `_source` will still remain same
    ```json
    // field level
    {
      "mappings": {
        "properties": {
          "rating": { "type": "float", "null_value": false }
        }
      }
    }
3. `copy_to` - used when we want to copy a value e.g. first_name and last_name. It won't be showed in _source object
    ```json
    // field level
    {
      "mappings": {
        "properties": {
          "first_name": { "type": "float", "copy_to": "full_name" },
          "last_name": { "type": "float", "copy_to": "full_name" }
        }
      }
    }
#### APIs
1. PUT /reviews - creating a new mapping 
```json
//field names as keys and objects as values
// body
{
  "mappings": {
    "properties": {
      "rating": { "type": "float"},
      "content": { "type": "text"},
      "product_id": { "type": "integer"},
      "author": { 
        "properties": {
          "first_name": {"type": "text"},
          "last_name": {"type": "text"},
          "email": {"type": "keyword"}
        }}
    }
  }
}
// body(dot notation)
{
  "mappings": {
    "properties": {
      "rating": { "type": "float"},
      "content": { "type": "text"},
      "product_id": { "type": "integer"},
      "author.first_name": {"type": "text"},
      "author.last_name": {"type": "text"},
      "author.email": {"type": "keyword"}
    }
  }
}
```
2. GET /reviews/_mapping - retrieving mapping for an index
```json
//results
{
  "reviews" : {
    "mappings" : {
      "properties" : {
        "author" : {
          "properties" : {
            "email" : {
              "type" : "keyword"
            },
            "first_name" : {
              "type" : "text"
            },
            "last_name" : {
              "type" : "text"
            }
          }
        },
        "content" : {
          "type" : "text"
        },
        "context" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "product_id" : {
          "type" : "integer"
        },
        "rating" : {
          "type" : "float"
        }
      }
    }
  }
}
```
3. GET /reviews/_mapping/field/content - retrieving mapping for a field  
GET /reviews/_mapping/field/author.email - retrieving mapping for a nested field
```json
//results
{
  "reviews" : {
    "mappings" : {
      "content" : {
        "full_name" : "content",
        "mapping" : {
          "content" : {
            "type" : "text"
          }
        }
      }
    }
  }
}
```
4. PUT /reviews/_mapping - adding a field to existing mappings
```json
//body
{
  "properties": {
    "created_at": {"type": "date"}
  }
}
```
5. POST /_reindex - reindexing from one index to another, alternatively script can be added while reindexing
```json
//body
{
  "source": {
    "index": "reviews",
    "query": { // optional field, a query can be specified to only match certain document
      "match_all": {},
      "rating": {
        "gte": 4.0
      }
    }
  },
  "dest": {
    "index": "reviews_new"
  },
  "script": {
    "source": """
      if (ctx._source.product_id != null) {
        ctx._source.product_id = ctx._source.product_id.toString();
      }      
    """"
  }
}
```
```json
//body (removing a field)
{
  "source": {
    "index": "reviews",
    "_source": ["content", "created_at", "rating"]
  },
  "dest": {
    "index": "reviews_new"
  }
}
```
```json
//body (renaming a field)
{
  "source": {
    "index": "reviews",
  },
  "dest": {
    "index": "reviews_new"
  },
  "script": {
    "source": """ctx._source.comment = ctx._source.remove(content);"""
  }
}
```
```json
//body (renaming a field)
{
  "source": {
    "index": "reviews",
  },
  "dest": {
    "index": "reviews_new"
  },
  "script": {
    "source": """
      if (ctx._source.rating < 4.0) {
        ctx.op = "noop"; # can also be set to "delete"
      }
    """
  }
}
```
6. PUT /reviews - adding field alias, so we can search on different field without reindexing. When we want to rename the field without reindexing. Can be updated
```json
//body
{
  "properties": {
    "comment": {"type": "alias", "path":  "content"}
  }
}
```
7. PUT /multi_field_test - adding more fields, `keyword` for indexing
```json
//body
{
  "mappings": {
    "properties": {
      "description": {
        "type": "text"
      },
      "ingredients": {
        "type": "text",
        "fields": {
          "keyword": {  // can be queried
            "type": "keyword"
          }
        }
      }
    }
  }
}

//query
GET /multi_field_test/_search
{
  "query": {
    "term": {
      "ingredients.keyword": "Spaghetti"
    }
  }
}
```
8. Index templates - assigning an index pattern, it will match the index and use the template matching. The settings from the new index will overwrite the ones in the template. If it matches to more than 1, all templates will be merged from lowest `order` to highest order.  
 PUT /_template/access-logs  
GET /_template/access-logs  
   DELETE /_template/access-logs
```json
PUT /_template/access-logs
{
  "index_patterns": ["access-logs-*"],
  "settings": {
    "number_of_shards": 2,
    "index.mapping.coerce": false
  }, 
  "mappings": {
    "properties": {
      "@timestamp": {
        "type": "date"
      },
      "url.original": {
        "type": "keyword"
      },
      "http.request.referrer": {
        "type": "keyword"
      },
      "http.response.status_code": {
        "type": "long"
      }
    }
  }
}
PUT /access-logs-2020-01-01
GET /access-logs-2020-01-01
GET /access-logs-2020-01-01
```

9. Setting Dynamic mapping - `false` will disable indexing the document, `strict` will reject the document
```json
PUT /people
//body
{
  "mappings": {
    "dynamic": "strict",  //can be false
    "properties": {
      "first_name": {
        "type": "text"
      }
    }
  }
}
```
10. Dynamic templates - set settings for dynamic mapping, this will map whle numbers to integer
```json
PUT /dynamic_template_test
{
  "mappings": {
    "dynamic_templates": [
      {
        "integers": {
          "match_mapping_type": "long",
          "mapping": {
            "type": "integer"
          }
        }
      }
    ]
  }
}
```
```json
PUT /test_index
//body this will match to string and make it a text field
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_only_text": {
          "match_mapping_type": "string",
          "match": "text_*",
          "unmatch": "*_keyword",
          "mapping": {
            "type": "text"
          }
        }
      },
      {
        "strings_only_keyword": {
          "match_mapping_type": "string",
          "match": "*_keyword",
          "mapping": {
            "type": "keyword"
          }
        }
      }
    ]
  }
}

POST /test_index/_doc
{
  "text_product_description": "A description.",
  "text_product_id_keyword": "ABC-123"
}
```
#### Using path_match
```json
PUT /test_index
{
  "mappings": {
    "dynamic_templates": [
      {
        "copy_to_full_name": {
          "match_mapping_type": "string",
          "path_match": "employer.name.*",
          "mapping": {
            "type": "text",
            "copy_to": "full_name"
          }
        }
      }
    ]
  }
}

POST /test_index/_doc
{
  "employer": {
    "name": {
      "first_name": "John",
      "middle_name": "Edward",
      "last_name": "Doe"
    }
  }
}
```
#### Using placeholder
```json
PUT /test_index
{
  "mappings": {
    "dynamic_templates": [
      {
        "no_doc_values": {
          "match_mapping_type": "*",
          "mapping": {
            "type": "{dynamic_type}",
            "index": false
          }
        }
      }
    ]
  }
}

POST /test_index/_doc
{
  "name": "John Doe",
  "age": 26
}
```
### Searching
The algorithm for relevance score used previously was TF/IDF before being changed to BM25 algorithm(Nonlinear term frequency saturation, stopwords will constantly get boost while there's a cap to BM25). You can do queries using cURL or body. The score are computed on a per shard basis hence if a document is the only results on a shard it may skew the results. On most occasion this is not a problem.  

We can filter results by query/filter. Query will consider the score in relevance while filter is either yes or no. Another point is that elastic search can cache filters.  
`GET /products/_search?q=name:Milk Frother AND price:129`
1. GET /products/_search?explain - adding `explain` to the end will explain the results
```json
//body (term level query search for exact matches in index and are not analyzed)
{
  "query": {
    "term": {
      "name": "milk" // must be normalized
    }
  }
}

//body (match level goes through the same analysis process before being compared)
{
  "query": {
    "match": {
      "name": "Milk Frother"
    }
  }
}
```
2. GET /products/_explain/201 - alternatively we can use this to crosscheck why the query match/doesn't match to the document id `201`
```json
// body
{
  "query": {
    "term": {
      "name": "Milk Frother"}
  }
}

//results
{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "201",
  "matched" : false,
  "explanation" : {
    "value" : 0.0,
    "description" : "no matching term",
    "details" : [ ]
  }
}
```
#### Term level Queries
This should be used only for enums, dates, numbers etc. Because term level queries search for exact matches.
1. GET /products/_search - search api supports 
   1. single term search
   2. multi term search
   3. searching in IDs
   4. searching range (date, numbers etc.)
   5. anchor dates(days from a given date e.g. 7 days from today) https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#date-math
   6. matching non-null values - check for field that exists
   7. matching based on prefixes
   8. matching wildcard queries - `?` matches any single character and `*` matches any sequences, is slow, if question mark or asterisk is placed on beginning, it will be slow
   9. Searching with regex queries - `regexp` field, this uses lucene regex so normal regex such as `/d` will not work. https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-regexp-query.html#regexp-syntax
```json
//body
{
  "query": {
    "term": {
      "is_active": true
    }
  }
}

//body (alternative way of writing queries)
{
  "query": {
    "term": {
      "is_active": {
         "value": true
      }
    }
  }
}
```
```json
//body (search multiple terms, will match any of the terms, this is an IN clause)
{
   "query": {
      "terms": {
         "tags.keyword": [ "Soup", "Cake" ] // will return with [Soup], [Cake] or [Soup, Cake]
      }
   }
}
```
```json
//body will return document with id 1,2,3
{
   "query": {
      "ids": {
         "values": [ 1, 2, 3 ]
      }
   }
}
```
```json
//Matching documents with an in_stock field of between 1 and 5, both included
{
   "query": {
      "range": {
         "in_stock": {
            "gte": 1,
            "lte": 5
         }
      }
   }
}

//Matching documents with a date range
{
   "query": {
      "range": {
         "created": {
            "gte": "2010/01/01",
            "lte": "2010/12/31"
         }
      }
   }
}

//Matching documents with a date range and custom date formatv
{
   "query": {
      "range": {
         "created": {
            "gte": "01-01-2010",
            "lte": "31-12-2010",
            "format": "dd-MM-yyyy"
         }
      }
   }
}
```
```json
//Subtracting one year from 2010/01/01
{
   "query": {
      "range": {
         "created": {
            "gte": "2010/01/01||-1y"
         }
      }
   }
}

//Subtracting one year and one day from 2010/01/01
{
   "query": {
      "range": {
         "created": {
            "gte": "2010/01/01||-1y-1d"
         }
      }
   }
}

//Subtracting one year from 2010/01/01 and rounding by month
{
   "query": {
      "range": {
         "created": {
            "gte": "2010/01/01||-1y/M"  # rounding off by month M
         }
      }
   }
}

//Rounding by month before subtracting one year from 2010/01/01
{
   "query": {
      "range": {
         "created": {
            "gte": "2010/01/01||/M-1y"  # the order is important, it will round before subtracting
         }
      }
   }
}

//Rounding by month before subtracting one year from the current date
{
   "query": {
      "range": {
         "created": {
            "gte": "now/M-1y"  # calculating date relative to current time
         }
      }
   }
}

//Matching documents with a created field containing the current date or later
{
   "query": {
      "range": {
         "created": {
            "gte": "now"  # will match all documents gte current time
         }
      }
   }
}
```
```json
//Matching documents with non-null values
{
  "query": {
    "exists": {
      "field": "tags"
    }
  }
}
```
```json
//Matching documents containing a tag beginning with Vege
{
   "query": {
      "prefix": {
         "tags.keyword": "Vege"
      }
   }
}
```
```json
//Adding an asterisk for any characters (zero or more)
{
   "query": {
      "wildcard": {
         "tags.keyword": "Veg*ble"
      }
   }
}

//Adding a question mark for any single character, this will not return any results
{
   "query": {
      "wildcard": {
         "tags.keyword": "Veg?ble"
      }
   }
}

//Adding a question mark for any single character
{
   "query": {
      "wildcard": {
         "tags.keyword": "Veget?ble"
      }
   }
}
```
```json
//Searching with regular expressions
{
   "query": {
      "regexp": {
         "tags.keyword": "Veg[a-zA-Z]+ble"
      }
   }
}
```
#### Full Text Queries
This should be used only for enums, dates, numbers etc. Because term level queries search for exact matches.
1. GET /recipe/_search
   1. `match` - it's a boolean word so any words in your search query that appeared will be shown as a result
   1. `match` - specifying `and` boolean operator, order does not matter or if there's any word in-between
   1. `match_phrase` - must be an exact phrase match, order matters
   1. `multi_match` - define an array of field to search through, the relevance score will be scored based on the field with most matching https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html#multi-match-types
```json
// Standard `match` query
{
  "query": {
    "match": {
      "title": "Recipes with pasta or spaghetti"
    }
  }
}
```
```json
// Specifying a boolean operator
{
   "query": {
      "match": {
         "title": {
            "query": "pasta or spaghetti",
            "operator": "and"
         }
      }
   }
}

// Specifying a boolean operator
{
   "query": {
      "match": {
         "title": {
            "query": "pasta spaghetti",  # this will still return same results as above because it's still a term level query
            "operator": "and"
         }
      }
   }
}
```
```json
//Searching multiple fields
{
   "query": {
      "multi_match": {
         "query": "pasta",
         "fields": [ "title", "description" ]  # will search pasta in title field/description field
      }
   }
}
```
### Underlying functionality for concurrency control
1. Primary terms - a way to distinguished between old and new primary shards, it is appended to write query. Allows the replica to know if whether the primary shard has changed since the write request is sent.
2. Sequence number - appended together with primary term, a counter that is incremented for each write operation, allowing es to know the order of write operations
3. Global checkpoints  - global checkpoints is the seq number that all active shards within a replication group has been aligned at least up to, local checkpoints is the sequence number for the last write operation that was performed. This means if an es reconnectes, it only need to compare the operations that are above the global checkpoint.
4. Local checkpoints - if a replica shard fails, only a seq higher than local checkpoint need to be applied when it reconnects.