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

#### Querying with boolean logic
Boolean logic is the internal workings of combining two queries into one using `AND` logic and `OR` logic
```json
// Adding query clauses to the `must` key
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "ingredients.name": "parmesan"
          }
        },
        {
          "range": {
            "preparation_time_minutes": {
              "lte": 15
            }
          }
        }
      ]
    }
  }
}

// Moving the range query to the `filter` key
{
   "query": {
      "bool": {
         "must": [
            {
               "match": {
                  "ingredients.name": "parmesan"
               }
            }
         ],
         "filter": [
            {
               "range": {
                  "preparation_time_minutes": {
                     "lte": 15
                  }
               }
            }
         ]
      }
   }
}

// Adding a query clause to the `must_not` key
{
   "query": {
      "bool": {
         "must": [
            {
               "match": {
                  "ingredients.name": "parmesan"
               }
            }
         ],
         "must_not": [
            {
               "match": {
                  "ingredients.name": "tuna"
               }
            }
         ],
         "filter": [
            {
               "range": {
                  "preparation_time_minutes": {
                     "lte": 15
                  }
               }
            }
         ]
      }
   }
}

// Adding a query clause to the `should` key
{
   "query": {
      "bool": {
         "must": [
            {
               "match": {
                  "ingredients.name": "parmesan"
               }
            }
         ],
         "must_not": [
            {
               "match": {
                  "ingredients.name": "tuna"
               }
            }
         ],
         "should": [
            {
               "match": {
                  "ingredients.name": "parsley"
               }
            }
         ],
         "filter": [
            {
               "range": {
                  "preparation_time_minutes": {
                     "lte": 15
                  }
               }
            }
         ]
      }
   }
}
```
#### How the match query works
When a match query is received, it will convert automatically to a boolean should query with analyzer applied. If you use a `bool -> must -> term` query, the analyzer will not be applied 
```json
{
  "query": {
    "match": {
      "title": "pasta carbonara"
    }
  }
}

// equivalent to
{
   "query": {
      "bool": {
         "should": [
            {
               "term": {
                  "title": "pasta"
               }
            },
            {
               "term": {
                  "title": "carbonara"
               }
            }
         ]
      }
   }
}
```
```json
{
  "query": {
    "match": {
      "title": {
        "query": "pasta carbonara",
        "operator": "and"
      }
    }
  }
}

// equivalent to
{
   "query": {
      "bool": {
         "must": [
            {
               "term": {
                  "title": "pasta"
               }
            },
            {
               "term": {
                  "title": "carbonara"
               }
            }
         ]
      }
   }
}
```
#### Joining queries/nested queries
Relationship database, nested query must be queried using the `nested` way. A nested query must contain:
1. a path to the field
2. a query to be run against
---
1. Adding Sample Datas  
```json
// Creating the index with mapping
PUT /department
{
   "mappings": {
      "properties": {
         "name": {
            "type": "text"
         },
         "employees": {
            "type": "nested"
         }
      }
   }
}

//adding test documents
PUT /department/_doc/1
{
   "name": "Development",
   "employees": [
      {
         "name": "Eric Green",
         "age": 39,
         "gender": "M",
         "position": "Big Data Specialist"
      },
      {
         "name": "James Taylor",
         "age": 27,
         "gender": "M",
         "position": "Software Developer"
      },
      {
         "name": "Gary Jenkins",
         "age": 21,
         "gender": "M",
         "position": "Intern"
      },
      {
         "name": "Julie Powell",
         "age": 26,
         "gender": "F",
         "position": "Intern"
      },
      {
         "name": "Benjamin Smith",
         "age": 46,
         "gender": "M",
         "position": "Senior Software Engineer"
      }
   ]
}

PUT /department/_doc/2
{
   "name": "HR & Marketing",
   "employees": [
      {
         "name": "Patricia Lewis",
         "age": 42,
         "gender": "F",
         "position": "Senior Marketing Manager"
      },
      {
         "name": "Maria Anderson",
         "age": 56,
         "gender": "F",
         "position": "Head of HR"
      },
      {
         "name": "Margaret Harris",
         "age": 19,
         "gender": "F",
         "position": "Intern"
      },
      {
         "name": "Ryan Nelson",
         "age": 31,
         "gender": "M",
         "position": "Marketing Manager"
      },
      {
         "name": "Kathy Williams",
         "age": 49,
         "gender": "F",
         "position": "Senior Marketing Manager"
      },
      {
         "name": "Jacqueline Hill",
         "age": 28,
         "gender": "F",
         "position": "Junior Marketing Manager"
      },
      {
         "name": "Donald Morris",
         "age": 39,
         "gender": "M",
         "position": "SEO Specialist"
      },
      {
         "name": "Evelyn Henderson",
         "age": 24,
         "gender": "F",
         "position": "Intern"
      },
      {
         "name": "Earl Moore",
         "age": 21,
         "gender": "M",
         "position": "Junior SEO Specialist"
      },
      {
         "name": "Phillip Sanchez",
         "age": 35,
         "gender": "M",
         "position": "SEM Specialist"
      }
   ]
}
```
```json
// body
GET /department/_search
{
   "query": {
      "nested": {
         "path": "employees",  # this will be the field of  the array
         "query": {
            "bool": {
               "must": [
                  {
                     "match": {
                        "employees.position": "intern"
                     }
                  },
                  {
                     "term": {
                        "employees.gender.keyword": {
                           "value": "F"
                        }
                     }
                  }
               ]
            }
         }
      }
   }
}

//results
{
   "took" : 114,
   "timed_out" : false,
   "_shards" : {
      "total" : 1,
      "successful" : 1,
      "skipped" : 0,
      "failed" : 0
   },
   "hits" : {
      "total" : {
         "value" : 2,
         "relation" : "eq"
      },
      "max_score" : 2.3905568,
      "hits" : [
         {
            "_index" : "department",
            "_type" : "_doc",
            "_id" : "2",
            "_score" : 2.3905568,
            "_source" : {
               "name" : "HR & Marketing",
               "employees" : [
                  {
                     "name" : "Patricia Lewis",
                     "age" : 42,
                     "gender" : "F",
                     "position" : "Senior Marketing Manager"
                  },
                  {
                     "name" : "Maria Anderson",
                     "age" : 56,
                     "gender" : "F",
                     "position" : "Head of HR"
                  },
                  {
                     "name" : "Margaret Harris",
                     "age" : 19,
                     "gender" : "F",
                     "position" : "Intern"
                  },
                  {
                     "name" : "Ryan Nelson",
                     "age" : 31,
                     "gender" : "M",
                     "position" : "Marketing Manager"
                  },
                  {
                     "name" : "Kathy Williams",
                     "age" : 49,
                     "gender" : "F",
                     "position" : "Senior Marketing Manager"
                  },
                  {
                     "name" : "Jacqueline Hill",
                     "age" : 28,
                     "gender" : "F",
                     "position" : "Junior Marketing Manager"
                  },
                  {
                     "name" : "Donald Morris",
                     "age" : 39,
                     "gender" : "M",
                     "position" : "SEO Specialist"
                  },
                  {
                     "name" : "Evelyn Henderson",
                     "age" : 24,
                     "gender" : "F",
                     "position" : "Intern"
                  },
                  {
                     "name" : "Earl Moore",
                     "age" : 21,
                     "gender" : "M",
                     "position" : "Junior SEO Specialist"
                  },
                  {
                     "name" : "Phillip Sanchez",
                     "age" : 35,
                     "gender" : "M",
                     "position" : "SEM Specialist"
                  }
               ]
            }
         },
         {
            "_index" : "department",
            "_type" : "_doc",
            "_id" : "1",
            "_score" : 2.3905568,
            "_source" : {
               "name" : "Development",
               "employees" : [
                  {
                     "name" : "Eric Green",
                     "age" : 39,
                     "gender" : "M",
                     "position" : "Big Data Specialist"
                  },
                  {
                     "name" : "James Taylor",
                     "age" : 27,
                     "gender" : "M",
                     "position" : "Software Developer"
                  },
                  {
                     "name" : "Gary Jenkins",
                     "age" : 21,
                     "gender" : "M",
                     "position" : "Intern"
                  },
                  {
                     "name" : "Julie Powell",
                     "age" : 26,
                     "gender" : "F",
                     "position" : "Intern"
                  },
                  {
                     "name" : "Benjamin Smith",
                     "age" : 46,
                     "gender" : "M",
                     "position" : "Senior Software Engineer"
                  }
               ]
            }
         }
      ]
   }
}
```

2. Inner Hits
```json
GET /department/_search
{
   "_source": false, # to hide the source document, just easier to read
   "query": {
      "nested": {
         "path": "employees",
         "inner_hits": {},  # adding this field
         "query": {
            "bool": {
               "must": [
                  {
                     "match": {
                        "employees.position": "intern"
                     }
                  },
                  {
                     "term": {
                        "employees.gender.keyword": {
                           "value": "F"
                        }
                     }
                  }
               ]
            }
         }
      }
   }
}

// results
{
   "took" : 3,
   "timed_out" : false,
   "_shards" : {
      "total" : 1,
      "successful" : 1,
      "skipped" : 0,
      "failed" : 0
   },
   "hits" : {
      "total" : {
         "value" : 2,  # how many items matches the search query
         "relation" : "eq"
      },
      "max_score" : 2.3905568,
      "hits" : [
         {
            "_index" : "department",
            "_type" : "_doc",
            "_id" : "2",
            "_score" : 2.3905568,
            "inner_hits" : {
               "employees" : {
                  "hits" : {
                     "total" : {
                        "value" : 2,  # how many nested objects matches the department
                        "relation" : "eq"
                     },
                     "max_score" : 2.3905568,
                     "hits" : [
                        {
                           "_index" : "department",
                           "_type" : "_doc",
                           "_id" : "2",
                           "_nested" : {
                              "field" : "employees",  # this tells us where the hit came from
                              "offset" : 2
                           },
                           "_score" : 2.3905568,
                           "_source" : {
                              "gender" : "F",
                              "name" : "Margaret Harris",
                              "position" : "Intern",
                              "age" : 19
                           }
                        },
                        {
                           "_index" : "department",
                           "_type" : "_doc",
                           "_id" : "2",
                           "_nested" : {
                              "field" : "employees",
                              "offset" : 7
                           },
                           "_score" : 2.3905568,
                           "_source" : {
                              "gender" : "F",
                              "name" : "Evelyn Henderson",
                              "position" : "Intern",
                              "age" : 24
                           }
                        }
                     ]
                  }
               }
            }
         },
         {
            "_index" : "department",
            "_type" : "_doc",
            "_id" : "1",
            "_score" : 2.3905568,
            "inner_hits" : {
               "employees" : {
                  "hits" : {
                     "total" : {
                        "value" : 1,
                        "relation" : "eq"
                     },
                     "max_score" : 2.3905568,
                     "hits" : [
                        {
                           "_index" : "department",
                           "_type" : "_doc",
                           "_id" : "1",
                           "_nested" : {
                              "field" : "employees",
                              "offset" : 3
                           },
                           "_score" : 2.3905568,
                           "_source" : {
                              "gender" : "F",
                              "name" : "Julie Powell",
                              "position" : "Intern",
                              "age" : 26
                           }
                        }
                     ]
                  }
               }
            }
         }
      ]
   }
}
```

3. Mapping Document relationships - to add relationship, the mapping need to be remade. We can use a special kind of field called `join` field. We can name the field anything we want. Possible relations must be defined in the `relations` object
```json
PUT /department
{
   "mappings": {
      "properties": {
         "join_field": {
            "type": "join",
            "relations": {
               "department_info": "employee_info"  # arbritary choice, key does not need to match index name, this defines a parent child relationship where `department` is the parent we can use this while adding documents, this field can be a children field by using arrays
            }
         }
      }
   }
}
```
Adding documents - we need to specify which relationship we want our document to be part of. Our relationship say `department` is parent of `employee`.  
The children  document is added to the same index, but with specification of the `join_field` and the related parent. In addition, we need to add a routing with the parent id info because parent+child id document must be stored in the same shard.
```json
PUT /department/_doc/1
{
  "name": "Development",
  "join_field": "department_info"  # this one should match the key name of the join field, with value matching the key of relationship name
}
PUT /department/_doc/2
{
   "name": "Marketing",
   "join_field": "department_info"
}
```
```json
PUT /department/_doc/3?routing=1  # if we use custom routing, we need to change this field accordingly
{
  "name": "Bo Andersen",
  "age": 28,
  "gender": "M",
  "join_field": {
    "name": "employee_info",
    "parent": 1
  }
}
```
```json
PUT /department/_doc/4?routing=2
{
  "name": "John Doe",
  "age": 44,
  "gender": "M",
  "join_field": {
    "name": "employee_info",
    "parent": 2
  }
}
```
```json
PUT /department/_doc/5?routing=1
{
  "name": "James Evans",
  "age": 32,
  "gender": "M",
  "join_field": {
    "name": "employee_info",
    "parent": 1
  }
}
```
```json
PUT /department/_doc/6?routing=1
{
  "name": "Daniel Harris",
  "age": 52,
  "gender": "M",
  "join_field": {
    "name": "employee_info",
    "parent": 1
  }
}
```
```json
PUT /department/_doc/7?routing=2
{
  "name": "Jane Park",
  "age": 23,
  "gender": "F",
  "join_field": {
    "name": "employee",
    "parent": 2
  }
}
```
```json
PUT /department/_doc/8?routing=1
{
  "name": "Christina Parker",
  "age": 29,
  "gender": "F",
  "join_field": {
    "name": "employee",
    "parent": 1
  }
}
```

4. Querying by parent ID - we need to tell es which relation we want to query and specify id of `parent_id`. The `hits` will have `join_field`. ES automatically use ID of parent ID to do routing, can be override if needed
```json
//GET /department/_search
{
  "query": {
    "parent_id": {  // query object
      "type": "employee",  // child
      "id": 1  // id of parent
    }
  }
}

//results
{
   "took" : 2,
   "timed_out" : false,
   "_shards" : {
      "total" : 1,
      "successful" : 1,
      "skipped" : 0,
      "failed" : 0
   },
   "hits" : {
      "total" : {
         "value" : 2,
         "relation" : "eq"
      },
      "max_score" : 0.8266786,
      "hits" : [
         {
            "_index" : "department",
            "_type" : "_doc",
            "_id" : "4",
            "_score" : 0.8266786,
            "_routing" : "2",
            "_source" : {
               "name" : "John Doe",
               "age" : 44,
               "gender" : "M",
               "join_field" : {
                  "name" : "employee_info",
                  "parent" : 2
               }
            }
         },
         {
            "_index" : "department",
            "_type" : "_doc",
            "_id" : "7",
            "_score" : 0.8266786,
            "_routing" : "2",
            "_source" : {
               "name" : "Jane Park",
               "age" : 23,
               "gender" : "F",
               "join_field" : {
                  "name" : "employee_info",
                  "parent" : 2
               }
            }
         }
      ]
   }
}
```

5. Query child documents by parent criteria(without knowing ID, using search criteria). Using `has_parent` key. We need to specify the `parent_type` and the query the parent document must match. Adding `score` can include relevance of results. Can be used incase it matches more than 1 document such as searching text fields.
```json
GET /department/_search
{
  "query": {
    "has_parent": {
      "parent_type": "department_info",
      "query": {  // conditions that parent document must satisfy
        "term": {
          "name.keyword": "Development"
        }
      }
    }
  }
}

//Incorporating the parent documents' relevance scores
GET /department/_search
{
   "query": {
      "has_parent": {
         "parent_type": "department",
         "score": true,
         "query": {
            "term": {
               "name.keyword": "Development"
            }
         }
      }
   }
}
```

6. Query parent by child documents criteria(without knowing ID, using search criteria). Using `has_child` key. We need to specify the `parent_type` and the query the parent document must match. There are 5 different score modes:
   1. min, lowest score of matching child is mapped to parent's relevant score
   2. max, highest score of matching child is mapped to parent's relevant score
   3. sum, total score of matching child is mapped to parent's relevant score
   4. avg, average score of matching child is mapped to parent's relevant score
   5. none, score of matching child is not considered  
The `has_child` can also add `min_children` and `max_children` to add a criteria. you can also add `sorting. We can add `inner_hits` to see how a document is being hit
```json
GET /department/_search
{
   "query": {
      "has_child": {
         "type": "employee_info",
         "score_mode": "sum",
         "query": {
            "bool": {
               "must": [
                  {
                     "range": {
                        "age": {
                           "gte": 50
                        }
                     }
                  }
               ],
               "should": [
                  {
                     "term": {
                        "gender.keyword": "M"
                     }
                  }
               ]
            }
         }
      }
   }
}

//Incorporating the parent documents' relevance scores
GET /department/_search
{
   "query": {
      "has_child": {
         "type": "employee",
         "inner_hits": {}, // to see which documents are being hit, optional
         "score_mode": "sum",
         "min_children": 2,
         "max_children": 5,
         "query": {
            "bool": {
               "must": [
                  {
                     "range": {
                        "age": {
                           "gte": 50
                        }
                     }
                  }
               ],
               "should": [
                  {
                     "term": {
                        "gender.keyword": "M"
                     }
                  }
               ]
            }
         }
      }
   }
}
```

7. Multi-level relations - do note that for this one you need to route it to the shard that contain the highest document of priority
```json
//Company -> Department -> Employee
//        -> Supplier
PUT /company
{
  "mappings": {
    "properties": {
      "join_field": { 
        "type": "join",
        "relations": {
          "company": ["department", "supplier"],
          "department": "employee"
        }
      }
    }
  }
}

// adding a company
PUT /company/_doc/1
{
   "name": "My Company Inc.",
   "join_field": "company"
}
```
```json
//Adding a company
PUT /company/_doc/1
{
   "name": "Development",
   "join_field": {
      "name": "department",
      "parent": 1
   }
}

//Adding a department
PUT /company/_doc/2?routing=1
{
   "name": "Development",
   "join_field": {
      "name": "department",
      "parent": 1
   }
}
        
//Adding an employee
PUT /company/_doc/3?routing=1
{
   "name": "Bo Andersen",
   "join_field": {
      "name": "employee",
      "parent": 2
   }
}
```
```json
// adding data
PUT /company/_doc/4
{
   "name": "Another Company, Inc.",
   "join_field": "company"
}
PUT /company/_doc/5?routing=4
{
   "name": "Marketing",
   "join_field": {
      "name": "department",
      "parent": 4
   }
}
PUT /company/_doc/6?routing=4
{
   "name": "John Doe",
   "join_field": {
      "name": "employee",
      "parent": 5
   }
}
// querying multi level
GET /company/_search
{
   "query": {
      "has_child": {
         "type": "department",
         "query": {
            "has_child": {
               "type": "employee",
               "inner_hits": {}, // to see which documents are being hit, optional
               "query": {
                  "term": {
                     "name.keyword": "John Doe"
                  }
               }
            }
         }
      }
   }
}
```
8. Querying Stories from a user's followers(get users, check the fields, get the documents in the fields). This is to show stories written by users he/she is following.
```json
// Sample user document
PUT /users/_doc/1
{
   "name": "John Roberts",
   "following" : [2, 3]
}

// sample stories document
PUT /stories/_doc/1
{
   "user_x": 3,
   "content": "Wow look, a penguin!"
}
PUT /stories/_doc/6
{
   "user_x": 2,
   "content": "Chilling at the beach @ Greece #vacation #goodtimes"
}

// get query
GET /stories/_search
{
  "query": {
    "terms": {
      "user_x": {  // get the results then put the results here
        "index": "users",
        "id": "1",
        "path": "following"
      }
    }
  }
}

//results
{
   "took" : 517,
   "timed_out" : false,
   "_shards" : {
      "total" : 1,
      "successful" : 1,
      "skipped" : 0,
      "failed" : 0
   },
   "hits" : {
      "total" : {
         "value" : 2,
         "relation" : "eq"
      },
      "max_score" : 1.0,
      "hits" : [
         {
            "_index" : "stories",
            "_type" : "_doc",
            "_id" : "1",
            "_score" : 1.0,
            "_source" : {
               "user" : 3,
               "content" : "Wow look, a penguin!"
            }
         },
         {
            "_index" : "stories",
            "_type" : "_doc",
            "_id" : "6",
            "_score" : 1.0,
            "_source" : {
               "user" : 2,
               "content" : "Chilling at the beach @ Greece #vacation #goodtimes"
            }
         }
      ]
   }
}
```
### Controlling Query Results
1. Controlling how the output looks like
```json
// output results as yaml
GET /recipe/_search?format=yaml
{
    "query": {
      "match": { "title": "pasta" }
    }
}

//output results pretty printed
GET /recipe/_search?pretty
{
   "query": {
      "match": { "title": "pasta" }
   }
}
```
2. Source filtering - to include/exclude fields. It supports asterisk, mainly used to reduce network latency
```json
// Excluding the `_source` field altogether
GET /recipe/_search
{
   "_source": false,
   "query": {
      "match": { "title": "pasta" }
   }
}

//Only returning the `created` field
GET /recipe/_search
{
   "_source": "created",
   "query": {
      "match": { "title": "pasta" }
   }
}

//Only returning an object's key(nested)
GET /recipe/_search
{
   "_source": "ingredients.name",
   "query": {
      "match": { "title": "pasta" }
   }
}

//Returning all of an object's keys
GET /recipe/_search
{
   "_source": "ingredients.*",
   "query": {
      "match": { "title": "pasta" }
   }
}

//Returning the ingredients object with all keys, and the servings field
GET /recipe/_search
{
   "_source": [ "ingredients.*", "servings" ],
   "query": {
      "match": { "title": "pasta" }
   }
}

//Including all of the ingredients object's keys, except the name key
GET /recipe/_search
{
   "_source": {
      "includes": "ingredients.*",
      "excludes": "ingredients.name"
   },
   "query": {
      "match": { "title": "pasta" }
   }
}
```
3. Specifying the result:
   1. size - we can add `size` to url and also the `total` will be returned, useful for pagination. default return size is 10
   2. specifying an offset - using the `from` keyword, inclusive
```json
// using a query parameter
GET /recipe/_search?size=2
{
  "_source": false,
  "query": {
    "match": {
      "title": "pasta"
    }
  }
}

// using request body
GET /recipe/_search
{
   "_source": false,
   "size": 2,
   "query": {
      "match": {
         "title": "pasta"
      }
   }
}
```
```json
//Specifying an offset with the from parameter
GET /recipe/_search
{
   "_source": false,
   "size": 2,
   "from": 2,
   "query": {
      "match": {
         "title": "pasta"
      }
   }
}
```
4. Sorting  
   1. Soring by ascending order(implicitly)
   2. Sorting by descending order - need to add `_source` to be used for sorting
   3. Sorting by multiple fields
   4. Sorting by multivalue fields(array)
```json
//body sort by ascending order
GET /recipe/_search
{
  "_source": false,
  "query": {
    "match_all": {}
  },
  "sort": [
    "preparation_time_minutes"
  ]
}

//sort by descending order
GET /recipe/_search
{
   "_source": "created",
   "query": {
      "match_all": {}
   },
   "sort": [
      { "created": "desc" }
   ]
}

// sort by multiple fields
GET /recipe/_search
{
   "_source": [ "preparation_time_minutes", "created" ],
   "query": {
      "match_all": {}
   },
   "sort": [
      { "preparation_time_minutes": "asc" },
      { "created": "desc" }
   ]
}

//Sorting by multi-value fields
GET /recipe/_search
{
   "_source": "ratings",
   "query": {
      "match_all": {}
   },
   "sort": [
      {
         "ratings": {
            "order": "desc",
            "mode": "avg"  //avg, max, min, none
         }
      }
   ]
}
```
### Improving the Search Results
1. Searching with fuzziness - adding fuzziness to work with typo, max distance is 2, do note that it is on a per term basis. It uses levenshtein distance which consider deletion, addition transposition as one step. You can disable transposition
```json
//Searching with `fuzziness` set to `auto`
GET /products/_search
{
   "query": {
      "match": {
         "name": {
            "query": "l0bster",
            "fuzziness": "auto"
            //"fuzzy_transpositions": false, can be optionally disabled
         }
      }
   }
}
```
2. Fuzzy query - fuzzy query is not analyzed, this is not same as the above.
```json
GET /products/_search
{
  "query": {
    "fuzzy": {
      "name": {
        "value": "LOBSTER",
        "fuzziness": "auto"
      }
    }
  }
}
```

3. highlighting fields - an addiotional `highlight` field will be in the result response. It will also give a little bit of context before and after the highlighted terms, enclosed in `<em>` tags. ES's highlight is based on the underlying implementation of analyzed document so we should not try to replicate it. We can custom the highlight tags to use a different tags instead.
```json
//Highlighting matches within the description field
GET /highlighting/_search
{
   "_source": false,
   "query": {
      "match": { "description": "Elasticsearch story" }
   },
   "highlight": {
      "fields": {
         "description" : {}
      }
   }
}

// results
{
   "took" : 6481,
   "timed_out" : false,
   "_shards" : {
      "total" : 1,
      "successful" : 1,
      "skipped" : 0,
      "failed" : 0
   },
   "hits" : {
      "total" : {
         "value" : 1,
         "relation" : "eq"
      },
      "max_score" : 0.68324494,
      "hits" : [
         {
            "_index" : "highlighting",
            "_type" : "_doc",
            "_id" : "1",
            "_score" : 0.68324494,
            "_ignored" : [
               "description.keyword"
            ],
            "highlight" : {
               "description" : [
                  "Let me tell you a <em>story</em> about <em>Elasticsearch</em>.",
                  "Lots of well-known and established companies use <em>Elasticsearch</em>, and so should you!"
               ]
            }
         }
      ]
   }
}
```
```json
//body to wrap with a different highlighting tags
GET /highlighting/_search
{
  "_source": false,
  "query": {
    "match": { "description": "Elasticsearch story" }
  },
  "highlight": {
    "pre_tags": [ "<strong>" ],
    "post_tags": [ "</strong>" ],
    "fields": {
      "description" : {}
    }
  }
}
```
### Underlying functionality for concurrency control
1. Primary terms - a way to distinguished between old and new primary shards, it is appended to write query. Allows the replica to know if whether the primary shard has changed since the write request is sent.
2. Sequence number - appended together with primary term, a counter that is incremented for each write operation, allowing es to know the order of write operations
3. Global checkpoints  - global checkpoints is the seq number that all active shards within a replication group has been aligned at least up to, local checkpoints is the sequence number for the last write operation that was performed. This means if an es reconnectes, it only need to compare the operations that are above the global checkpoint.
4. Local checkpoints - if a replica shard fails, only a seq higher than local checkpoint need to be applied when it reconnects.