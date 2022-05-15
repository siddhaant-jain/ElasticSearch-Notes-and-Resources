- insert a document(json) in elastic search
    ```javascript
    PUT people/_doc/1
    {
    "name": "Siddhant",
    "location": "Jhansi"
    }
    PUT people/_doc/2
    {
        "name": "Sowmya",
        "location": "Tumkur"
    }
    ```
- here 'people' is like the table, _doc is a convention followed while inserting json document
- if a particular table doesn't exist, it will be created when we run the put query
- this can be customised and stop bcz in production it's not preferred to automatically create table
- 1 is the id of the document we're inserting 

- in elastic search it is not called table or folder but index
- inside index we have shards

### Index and shards
- index is a logical structure (buckets for some kind of topics). It is distributed in multiple shards.
- Shard is Lucene instance (an engine which holds the data). It is a physical instance unlike index
- each shard will also have replicas which are just copies of shards that help in read requests. Replaces are also lucene instances
- obviously shards and replicas are on separate nodes else if nodes is down then both primary and replica shards are gone
- each cluster have a health status which can be green, yellow or red. It the cluster is up but all the primary shards are down then it will be red, if primary shards are also up but replica shards are down then it'll be yellow and if everything is up then it'll be green

- the output of put looks like
    ```javascript
    {
        "_index" : "people",
        "_id" : "2",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
            "total" : 2,
            "successful" : 1,
            "failed" : 0
        },
        "_seq_no" : 1,
        "_primary_term" : 1
    }
    ```
- when we run put query for a doc id which is already present, result key will have value 'updated' and version will be incremented by 1

- get a document by its id
    ```javascript
    GET people/_doc/2
    ```
- output will look like
    ```javascript
    {
        "_index" : "people",
        "_id" : "2",
        "_version" : 2,
        "_seq_no" : 2,
        "_primary_term" : 1,
        "found" : true,
        "_source" : {
            "name" : "Sowmya",
            "location" : "Tumkur"
        }
    }
    ```
- actual data is contained in _source. rest all is metadata 
- but what we did above is not an actual search. this can be done with a simple db query

- we can use _search api of GET for this
    ```javascript
    GET people/_search
    {
        "query": {
            "match": {
            "name": "Siddhant"
            }
        }
    }
    ```
- this is a very simple example

- type of nodes 
    1. Master Node: Responsible for adding any new node that joins the cluster and manage all the nodes. It doesn't manage any incoming requests or sending responses
    2. Data node: Responsible for holding data
    3. Coordinator node: all nodes whether master node or data node can be coordinator nodes. Request comes to coordinator node and distributes it among all the nodes. It also gets response from all the nodes and sends it back
    4. ingest node

- each search query goes through analysis in seach engine where the query is optimised. For eg. words are converted to their root word (like playing or played will become play etc.) which makes query much more efficient.

- if we simply do GET in an index, we get all info about that index
    ```javascript
    GET people
    ```
- response will be a json with 'people' dictionary
- it will contain three elements: 'aliases', 'mappings', 'settings'
- setting will have info like number of shards, number of replicas
- mapping will have details of all columns in the document. (like a schema)
- it auto infers the schema, but we should avoid that and provide a schema beforehand only

- if we do a get on index which we have not created and doesn't have any PUT request (which also creates the index) then we can 'index_not_found_exception'

- now we'll create a index where we provide the schema (mappings) and change number of replicas and shards (settings)
    ```javascript
    PUT students 
    {
        "mappings": {
            "properties": {
            "name": {
                "type": "text"
            },
            "age": {
                "type": "integer"
            }
            }
        },
        "settings": {
            "number_of_shards": 2,
            "number_of_replicas": 10
        }
    }
    ```

- if we try to create a index using PUT (like above eg.) which already exists then it will give "resource_already_exists_exception"
- to recreate the index we can first delete it like:
    ```javascript
    DELETE students
    ```

- the above query is to delete the entire index. But we can also delete one or some of the documents in the index.

- to put a document in index we can use either put or post, when we use PUT we need to provide id of document also, but with POST it generates an id automatically
    ```javascript
    PUT students/_doc/1
    {
    "name": "Siddhant",
    "age": "25"
    }
    POST students/_doc
    {
    "name": "Sowmya",
    "age": "25"
    }

    GET students
    ```

- while creating an index we can provide an alias and both can be used alternatively when using put, post or get request later
    ```javascript
    PUT bachelor_of_technology {
        "aliases": {
            "btech" : {}
        }
    }
    ```
- same alias can be given to multiple indexes. This means that if we want to search something in 10 different indexes then we don't need to mention them individually if they all have same alias. We can simply search in that alias.

- two type of searches
    1. term-level search which we can easily do with db query like searching (where we are looking for eact values for a column) 
    2. Full text search (search to get relevent documents)

- there is a single api through which all the search happens: GET index_name/_search
- term level search is done using request uri type request which looks like GET index_name/_search?q=col_name:expected_value
    ```javascript
    GET covid/_search?q=country:India
    ```
- Full text search is done using Query DSL which looks like, GET index_name/_search {json_body}
- term level search can also be done using query DSL (and it is preferred)

- getting a specific set of ids
    ```javascript
    GET covid/_search
    {
    "query": {
        "ids": {
        "values": [1,4]
        }
    }
    }
    ```
- result will have a key 'hits' whose value will be a list of document. Each document will be the result of our search. Each document will have a key 'score' which will have a float value showing the relevance (confidence) of this document for the search
- search for a value in a columne
    ```javascript
    GET covid/_search
    {
    "query": {
        "term": {
        "critical": {
            "value": "2731"
        }
        }
    }
    }
    ```
- if we're searching a text field with 'term' then it should be all lower case and only a single word
    ```javascript
    GET covid/_search
    {
    "query": {
        "term": {
        "country": {
            "value": "united"
        }
        }
    }
    }
    ```
- the above query will return both USA and UK with different score values
- we cann add highlight to the query. there will be a 'highlight' key with each document which will contain the mentioned column with seach word enclosed in em tag to emphasise or bold it
    ```javascript
    GET covid/_search
    {
    "query": {
        "term": {
        "country": {
            "value": "united"
        }
        }
    },
    "highlight": {
        "fields": {
        "country": {}
        }
    }
    }
    ```

- instead of 'term' we can also use 'terms' to search multiple words. It will return documents which have any of the words.
    ```javascript
    GET covid/_search
    {
    "query": {
        "terms": {
        "country": ["united", "states", "india"]
        }
    },
    "highlight": {
        "fields": {
        "country": {}
        }
    }
    }
    ```
- above query will return USA and UK and India
- we can also use range queries on a field
    ```javascript
    GET covid/_search
    {
    "query": {
        "range": {
        "cases": {
            "gte": 10000000,
            "lte": 20000000
        }
        }
    }
    }
    ```
- this will return all countries with cases b/w 10 million and 20 million

- we can also do fuzzy query to search closest word to what we searched it can take values b/w 0 and 2. More the value, more different word it can cover
    ```javascript
    GET covid/_search
    {
    "query": {
        "fuzzy": {
        "country": {
            "value": "finddom",
            "fuzziness": 2.0
        }
        }
    },
    "highlight": {
        "fields": {
        "country": {}
        }
    }
    }
    ```
- above will return united kingdom with kingdom emphasised

- we can use match query (most used) to find values in text field. By defult it uses an or operator and seach for any of the mentioned words. So if we search for 'united', 'states' it will return both USA and UK since it sees it as 'united' or 'states'. We can change that using 'operator' keyword
    ```javascript
    GET covid/_search
    {
    "query": {
        "match": {
        "country": {
            "query": "united states",
            "operator": "and"
        }
        }
    },
    "highlight": {
        "fields": {
        "country": {}
        }
    }
    }
    ```
- above will only return USA
- we can add fuzziness to a match query as well (misspelt united in below query)
    ```javascript
    GET covid/_search
    {
    "query": {
        "match": {
        "country": {
            "query": "unated states",
            "operator": "and",
            "fuzziness": 1
        }
        }
    },
    "highlight": {
        "fields": {
        "country": {}
        }
    }
    }
    ```
- we can use match_phrase query to find an exact substring in a column. The words cannot be distributed separately in the text the should be coming together the same way we have mentioned in query.
- if we want to make the match_phrase query a little flexible where if a certain number of words are missing in phrase but present in document they should also be displayed we can do that using 'slop' keyword which will take an integer values as to number of words that can be missing
- to get all the documents
    ```javascript
    GET covid/_search
    {
    "query": {
        "match_all": {}
    }
    }
    ```