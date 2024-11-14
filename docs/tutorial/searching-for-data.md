---
sidebar_position: 3
---

# Searching for Data

## Introduction to searching

- There are two ways of searching;
- **Request URI**
  - Search queries are added to the URL
  - Uses Apache Lucene’s query syntax
  - Only supports relatively simple queries
- **Query DSL**
  - Search queries are defined as JSON within the request body
  - More verbose, but supports all features
- Query DSL Example:

```SQL
GET /products/_search
{
    "query": {
        "match_all": {}
    }
}
```

## Introduction to term level queries

- One group of Elasticsearch queries is called **term level queries**
- Used to search structured data for exact values (filtering)
  - E.g. finding products where the brand equals "Nike"
- **Term level queries are not analyzed**
  - The search value is used exactly as is for inverted index lookups
- Can be used with data types such as **keyword**, numbers, dates, etc.

:::warning
Term level queries are case sensitive
:::

:::danger
Just don’t use term level queries for **text** fields!

e.g A query for “nike” works fine, but “Nike” doesn’t match anything
:::

## Searching for terms

- One of the most important search queries in Elasticsearch
- Used to query several different data types
- Text values (`keyword` only!), numbers, dates, booleans, ...
- Case sensitive by default
  - A `case_insensitive` parameter was added in v7.1.0
- Use the `terms` query to search for multiple terms

```SQL
GET /products/_search
{
    "query": {
        "term": {
            "tags.keyword": "Vegetable"
        }
    }
}
```

- Parameter that allows to perform case insensitive searches

```SQL
GET /products/_search
{
    "query": {
        "term": {
            "tags.keyword": "Vegetable",
            "case_insensitive": true
        }
    }
}
```

- Search for multiple items

```SQL
GET /products/_search
{
    "query": {
        "terms": {
            "tags.keyword": ["Soup", "Meat"]
        }
    }
}
```

## Retrieving documents by IDs

Example:

```SQL
GET /products/_search
{
    "query": {
        "ids": {
            "values": ["100", "200", "300"]
        }
    }
}
```

is equivalent to this in SQL:

`SELECT * FROM products WHERE _id IN ("100", "200", "300");`

## Range Searches

- The `range` query is used to perform range searches
- E.g. `in_stock >= 1` and `in_stock <= 5`
- E.g. `created >= 2020/01/01` and `created <= 2020/01/31`

### Querying Numeric Ranges

- Products that are almost sold out example

```SQL
GET /products/_search
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
```

is equivalent to this in SQL:

`WHERE in_stock >= 1 AND in_stock <= 5`

- Boundaries not included

```SQL
GET /products/_search
{
    "query": {
        "range": {
            "in_stock": {
                "gt": 1,
                "lt": 5
            }
        }
    }
}
```

is equivalent to this in SQL:

`WHERE in_stock > 1 AND in_stock < 5`

### Querying Dates with timestamps

- Use the `range` query to perform range searches
- Specify one or more of the `gt`, `gte`, `lt`, or `lte` parameters
- Supports both numbers and dates
- Dates are automatically handled for `date` fields
  - Specifying the time is optional, but recommended if possible
  - Custom formats are supported through the `format` parameter
  - Time zones are handled with the `time_zone` parameter (UTC offset)

```SQL
GET /products/_search
{
    "query": {
        "range": {
            "created": {
                "time_zone": "+01:00",
                "format": "yyyy/MM/dd",
                "gte": "2020/01/01 00:00:00",
                "lte": "2020/01/31 23:59:59"
            }
        }
    }
}
```

## Prefixes, wildcards, & regular expressions

- Term level queries are used for exact matching
  - Query non-analyzed values with queries that are not analyzed
- There are a few exceptions
  - Querying by prefix, wildcards, and regular expressions
  - Remember to still query `keyword` fields
- The `prefix` query matches terms that begin with a prefix
- The `wildcard` query enables us to use wildcards
- `?` to match any single character
- `*` to match any number of characters (`0-N`)
- Avoid placing wildcards at the beginning of patterns if at all possible
- Use the `"case_insensitive": true` parameter to ignore letter casing

**Querying by prefix Example**

```SQL
GET /products/_search
{
    "query": {
        "prefix": {
            "name.keyword": {
                "value": "Past"
            }
        }
    }
}
```

**Querying by wildcard Example**

```SQL
GET /products/_search
{
    "query": {
        "prefix": {
            "name.keyword": {
                "value": "Past?"
            }
        }
    }
}
```

**Querying by wildcard Example**

```SQL
GET /products/_search
{
    "query": {
        "prefix": {
            "name.keyword": {
                "value": "Bee*"
            }
        }
    }
}
```

### Regular Expressions

- The `regexp` query matches terms that match a regular expression
- Regular expressions are patterns used for matching strings
- Allows more complex queries than the `wildcard` query
- The whole term must be matched
- Uses Apache Lucene regex engine (`^` and `$` anchors not supported)

## Querying by field existence

- The `exists` query matches fields that have an indexed value
- Field values are only indexed if they are considered non-empty

```SQL
GET /products/_search
{
    "query": {
        "exists": {
            "field": "tags.keyword"
        }
    }
}
```

### Reasons for no indexed value

- Empty value provided (`NULL` or `[]`)
  - The `null_value` parameter is an exception for `NULL` values
- No value was provided for the field
- The `index` mapping parameter is set to `false` for the field
- The value’s length is greater than the `ignore_above` parameter
- Malformed value with the `ignore_malformed` mapping parameter set to `true`

### Inverting the query

- The `exists` query can be inverted by using the `bool` query’s `must_not` occurrence type

Example:

```SQL
GET /products/_search
{
    "query": {
        "bool": {
            "must_not": [
                {
                    "exists": {
                        "field": "tags.keyword"
                    }
                }
            ]
        }
    }
}
```

is equivalent to in SQL:

`SELECT * FROM products WHERE tags is NULL`

## Introduction to full text queries

- Term level queries are used for exact matching on structured data
- Full text queries are used for searching unstructured text data
  - E.g. website content, news articles, emails, chats, transcripts, etc.
  - Often used for long texts
  - We don’t know which values a field may contain (hence “unstructured”)
- Full text queries are not used for exact matching
  - They match values that include a term, often being one of many

:::note
Full text queries are analyzed with the field mapping's analyzer.

The resulting term is used for a loopup within the inverted index.
:::

### Full text queries vs term level queries

- The main difference is that full text queries are analyzed
  - Term level queries aren’t and are therefore used for exact matching

:::warning

- Don’t use full text queries on `keyword` fields because the field values were not analyzed during indexing
  - That compares analyzed values with non-analyzed values

:::

## The `match` query

- The `match` query is a fundamental query in Elasticsearch
- The most widely used full text query
- Powerful & flexible when using advanced parameters
- Supports most data types (e.g. dates and numbers)
  - Recommendation: Use term level queries if you know the input value
- If the analyzer outputs multiple terms, at least one must match by default
  - This can be changed by setting the `operator` parameter to `"and"`
- Matches documents that contain one or more of the specified terms
- The search term is analyzed and the result is looked up in the field’s
  inverted index

```SQL
GET /products/_search
{
    "query": {
        "match": {
            "name": "PASTA CHICKEN"
        }
    }
}
```

:::note
`"PASTA CHICKEN"` --ANALYZER--> `["pasta", "chicken"]` --> `"pasta"` OR `"chicken"`
:::

- Explicitly say pasta AND chicken Example:

```SQL
GET /products/_search
{
    "query": {
        "match": {
            "name": "PASTA CHICKEN"
            "operator": "AND"
        }
    }
}
```

## Introduction to relevance scoring

- Query results are sorted descendingly by the \_score metadata field
  - A floating point number of how well a document matches a query
- Documents matching term level queries are generally scored 1.0
  - Either a document matches, or it doesn’t (simply filtered out)
- Full text queries are not for exact matching
  - How well a document matches is now a factor
  - The most relevant results are placed highest (e.g. like on Google)

## Searching Multiple Fields

- [Multi-match Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html)
- The `multi_match` query performs full text searches on multiple fields
  - A document matches if at least one field is matched
- Individual fields can be relevance boosted by modifying the field name (^)
- Internally, Elasticsearch rewrites the query to simplify things for us
- By default, the best matching field’s relevance score is used for the document
  - Can be configured with the `type` parameter

```SQL
GET /products/_search
{
    "query": {
        "multi-match": {
            "query": "vegetable"
            "fields": ["name", "tags"]
        }
    }
}
```

- Relevance Boost Documents

```SQL
GET /products/_search
{
    "query": {
        "multi-match": {
            "query": "vegetable"
            "fields": ["name^2", "tags"]
        }
    }
}
```

### Specifying a tie breaker

- By default, one field is used for calculating a document’s relevance score
- We can “reward” documents where multiple fields match with the `tie_breaker` parameter
  - Each matching field affects the relevance score

## Phrase searches

- The `match_phrase` query is similar to the `match` query in some ways
- For the `match_phrase` query, the position (and thereby order) of terms matters
- Terms must appear in the correct order and with no other terms in-between
- The `standard` analyzer’s tokenizer outputs term positions that are stored
  within the field’s inverted index
  - These positions are then used for phrase searches (among others)

```SQL
GET /products/_search
{
    "query": {
        "match-phrase": {
            "description": "Elasticsearch guide"
        }
    }
}
```

## Leaf and compound queries

- **Leaf queries** search for values and are independent queries
  - e.g. the `term` and `match` queries
- Compound queries wrap `other` queries to produce a result

## Querying with boolean logic

- [Boolean query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html)
- match queries are usually translated into bool queries internally

```SQL
GET /products/_search
{
    "query": {
        "bool": {
            "must": [
                {
                    "term": {
                        "tag.keyboard": "Alcohol"
                    }
                }
            ],
            "must_not": [
                {
                    "term": {
                        "tags.keyword": "Wine"
                    }
                }
            ],
            "should": [
                {
                "term": {
                        "tags.keyword": "Beer"
                    }
                },
                {
                    "match": {
                        "name": "beer"
                    }
                },
                {
                    "match": {
                        "description": "beer"
                    }
                }

            ]
        }
    }
}
```

### The `must` occurrence type

- Query clauses are required to match and will contribute to relevance scores

### The `must_not` occurrence type

- Query clauses must not match and do not affect relevance scoring.
- Query clauses may therefore be cached for improved perfomance.

### Important things about `should`

- If a `bool` query only contains `should` clauses, **at least one must match**
- Useful if you just want something to match and reward matching documents
  - If nothing were required to match, we would get irrelevant results
- If a query clause exists for `must`, `must_not`, or `filter`, no `should` clause is
  required to match
  - Any `should` clauses are only used to boost relevance scores
- `minimum_should_match` behavior enforces the must clause and any of the should clauses must match.

### The `filter` occurrence type

- Query clauses must match
- Similar to the `must` occurrence type
- Ignores relevance scores
  - This improves the performance of the query
  - Query results can be cached and reused

## Query execution contexts

- Answers two questions;
  - "Does this document match" (yes/no)
  - "How well does this document match" (`_score` metadata field)
- Query results are sorted by `_score` descendingly
  - The most relevant documents appear at the top
- The query execution context calculates relevance scores

### Filter execution context

- Only answers one question: "Does this document match?" (yes/no)
  - No relevance scores are calculated
- Used to filter data, typically on structured data (dates, numbers, `keyword`)
  - Relevance scoring is irrelevant if we just want to filter out documents
- Improves performance
  - No resources are spent calculating relevance scores
  - Query results can be cached

### Changing the execution context

- It’s sometimes possible to change the execution context
  - Only a few queries support it, though
- Typically done with the `bool` query and `filter` aggregation
- Queries that support this generally have a `filter` parameter

## `boosting` Query

- The `bool` query enables us to increase relevance scores with `should`
- The `boosting` query can decrease relevance scores with `negative`
  - Documents must match the `positive` query clause
  - Documents that match the `negative` query clause have their relevance scores decreased
  - Use `match_all` query for `positive` if you don’t want to filter documents
  - Can be used with any query (including compound queries, such as `bool`)

```SQL
GET /products/_search
{
    "size": 20,
    "query": {
        "boosting": {
            "positive": {
                "match": {
                    "name": "juice"
                }
            },
            "negative": {
                "match": {
                    "name": "apple"
                }
            },
            "negative_boost": 0.5
        }
    }
}
```

## Disjunction max (`dis_max`)

- The `dis_max` query is a compound query
  - A document matches if at least one leaf query matches
- The best matching matching query clause’s relevance score is used for a
  document’s `_score`
- `tie_breaker` can be used to “reward” documents that match multiple queries
- `multi_match` queries are often translated into `dis_max` queries internally

### dis_max query

- The best matching field’s relevance score is used

```SQL
GET /products/_search
{
    "query": {
        "dis_max": {
            "queries": [
                {
                    "match": {
                        "name": "vegetable"
                    }
                },
                {
                    "match": {
                        "tags": "vegetable"
                    }
                }
            ]
        }
    }
}
```

## Querying nested objects

- **Problem**:
  - When indexing arrays of objects, the relationships between values are not maintained
  - Queries can yield “unpredictable” results
- **Solution**:
- Use the `nested` data type if you want to query objects independently
  - Otherwise the relationships between object values are not maintained
  - Each nested object is indexed as a hidden Lucene document
- Use the `nested` query on fields with the `nested` data type
  - Elasticsearch then handles everything automatically
- Create a new index to update the field mapping & reindex documents
- Nested Example:

```SQL
GET /recipes/_search
{
    "query": {
        "nested": {
            "path": "ingredients",
            "query": {
                "bool":{
                    "must": [
                        {
                            "match": {
                                "ingredients.name": "parmesan"
                            }
                        },
                        {
                            "range": {
                                "ingredients.amount": {
                                    "gte": 100
                                }
                            }
                        }
                    ]
                }
            }
        }
    }
}
```

:::note
For the example, above suppose we have a list of different recipes with ingredients for each one.

The query above will prevent the issue of showing all recipes containing both parmesan and an ingredient count over 100 for any other ingredients.

The query will just show recipes containing parmesan of at least count 100.
:::

### How documents are stored

- Matching child objects affect the parent document's relevance score
- Elasticsearch calculates a relevance score for each matching child object
  - This is because each nested object is a Lucene document
- Use the `score_mode` parameter to adjust relevance scoring

## Nested inner hits

- With the `nested` query, matches are “root documents”
  - E.g. recipes when searching for ingredients
- Sometimes we might want to know what matched instead of just something
- Nested inner hits tell us which nested object(s) matched the query
  - E.g. which ingredient(s) matched in a recipe
- Without inner hits, we only know that something matched
- Simply add the `inner_hits` parameter to the `nested` query
- Supply `{}` as the value for the default behavior
- Information about the matched nested object(s) is added to search results
- Use the `offset` key to find each object's position within `_source`
- Customize results with the `name` and `size` parameters
- Example with `inner_hits`

```SQL
GET /recipes/_search
{
    "query": {
        "nested": {
            "path": "ingredients",
            "inner_hits": {
                "name": "my_hits",
                "size": 10
            },
            "query": {
                "bool":{
                    "must": [
                        {
                            "match": {
                                "ingredients.name": "parmesan"
                            }
                        },
                        {
                            "range": {
                                "ingredients.amount": {
                                    "gte": 100
                                }
                            }
                        }
                    ]
                }
            }
        }
    }
}
```

:::info

`inner_hits` can take two parameters:

1. `name` enables us to change the key that appears directly within the inner_hits
2. `size` enables us to configure how many inner hits we want to be returned for each matching document.

:::

## Nested fields limitations

### Performance

- Indexing and querying `nested` fields is more expensive than for other data types
  - Keep performance in mind when using `nested` fields
  - If you map documents well, you should be all good, though
  - Denormalizing data is often a good idea
  - Elasticsearch has a few settings to prevent things from going wrong
- An Apache Lucene document is created for each nested object
  - Increased storage & query costs
- Important to remember for large datasets
- Elasticsearch provides safeguards to reduce the risk of performance bottlenecks

### Limitations

- We need to use a specialized data type (`nested`) and query (`nested`)
- Max 50 `nested` fields per index
  - Can be increased with the `index.mapping.nested_fields.limit` setting (not recommended)
- 10,000 nested objects per document (across all `nested` fields)
  - Protects against out of memory (OOM) exceptions
  - Can be increased with the `index.mapping.nested_objects.limit` setting (not recommended)
