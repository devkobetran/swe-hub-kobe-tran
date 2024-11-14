---
sidebar_position: 7
---

# Improving Search Results

## Proximity Searches

```SQL
GET /proximity/_search
{
    "query": {
        "match_phrase": {
            "title": {
                "query": "spicy sauce",
                "slop": 1
            }
        }
    }
}
```

:::note
Suppose that we want to allow a term in between the spicy and sauce terms, we can accomplish this by adding a parameter named `slop` for the match phrase query.

The value for this slop parameter should be an integer representing how far apart terms are allowed to be while still being considered a match.

How far apart refers to how many times a term may be moved for a document to match.

This example above satisfies a slop value of one which allows a single move to be made to make a document match.

e.g. It allows "spicy tomato sauce" to match
:::

## Affecting Relevance scoring with proximity

- The closer the proximity, the higher the relevance scores.
  - In other words, the lower the edit distance, the higher the relevance scores.
- Proximity is one of the factors of relevance scores but is not the only one that makes a big difference.

```SQL
GET /proximity/_search
{
    "query": {
        "bool": {
            "must": [
                "match": {
                    "title": {
                        "query": "spicy sauce"
                    }
                }
            ],
            "should": [
                {
                    "match_phrase": {
                        "title": {
                            "query": "spicy sauce",
                            "slop": 5
                        }
                    }
                }
            ]
        }
    }
}
```

## Fuzzy Match Query (Handling Typos)

- This `fuzziness` parameters account for when users make typos or mispellings

```SQL
GET /product/_search
{
    "query": {
        "match": {
            "name": {
                "query": "l0bster",
                "fuzziness": "auto"
            }
        }
    }
}
```

- Without the `fuzziness` parameter, "l0bster" will not show "lobster" search results.
- Maximum fuzziness value is 2.
  - Most misspellngs are just 1 or 2 characters.

```SQL
GET /product/_search
{
    "query": {
        "match": {
            "name": {
                "query": "l0bster love",
                "operator": "and",
                "fuzziness": 1
            }
        }
    }
}
```

- fuzziness of 1 can also be applied for each word in the query.

```SQL
GET /product/_search
{
    "query": {
        "match": {
            "name": {
                "query": "lvie",
                "fuzziness": 1,
                "fuzzy_transpositions": false
            }
        }
    }
}
```

- Transpositions example: AB -- BA
- `fuzzy_transpositions` are enabled true by default.
  - They count as 1 edit instead of 2.
  - e.g. live -- lvie
  - If `fuzzy_transpositions` is false, need to set `fuzziness` to 2 to find search results with "live" value.

## Fuzzy Query

```SQL
GET /product/_search
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

- Fuzzy Query is a term-level query.
  - Thus, fuzzy query is not analyzed.

:::tip
Therefore, prefer to use the match query with a `fuzziness` parameter.
:::

## Adding Synonyms

```SQL
PUT /synonyms
{
    "settings": {
        "analysis": {
            "filter": {
                "synonym_test": {
                    "type": "synonym",
                    "synonyms": [
                        "awful" => "terrible",
                        "awesome" => "great, super",
                        "weird, strange"
                    ]
                }
            },

            ...
        }
    }
}
```

:::note
All "awfuls" being search will show up terrible, too.

"weird, strange" are placed at the same position, so no replacement occurs.
:::

## Adding synonyms from file

- For words with so many synonyms, store the synonyms on a text file to reference

```SQL
...

"synonym_test": {
    "type": "synonym",
    "synonyms_path": "analysis/synonyms.txt"
}

...
```

- The synonym files should be available on all notes storing documents for the index using the analyzer.

:::warning
Elasticsearch picks up the new synonym when restarting a node but it does not re-index documents when starting back up.
:::

## Highlighting matches in fields

```SQL
GET /highlighting/_search
{
    "_source": false,
    "query": {
        "match": { "description": "Elasticsearch story" }
    },
    "highlight": {
        "fields": {
            "description": {}
        }
    }
}
```

- Next example, wrap matches within a different tag.
  - Use pre-tags and post-tags parameters.

```SQL
GET /highlighting/_search
{
    "_source": false,
    "query": {
        "match": { "description": "Elasticsearch story" }
    },
    "highlight": {
        "pre-tags": ["<strong>"],
        "post-tags": ["</strong>"],
        "fields": {
            "description": {}
        }
    }
}
```

## Stemming

- Use stemming to improve the matches of search queries.
- A stemmer is applied to a field.
- Elasticsearch will highlight the original words even after stemming has been applied to a text field.

```SQL
PUT /stemming_test
{
    ...

    "stemmer_test": {
        "type": "stemmer",
        "name" : "english"
    }
}
```
