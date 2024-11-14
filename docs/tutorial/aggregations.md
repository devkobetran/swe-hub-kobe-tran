---
sidebar_position: 6
---

# Aggregations

## Metric Aggregations

- [Metric Aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-metrics.html)
- Single-value numeric metric aggregations outputs a single value
- Multi-value metric aggregations outputs multiple values

```SQL
GET /order/_search
{
    "size": 0,
    "aggs": {
        "total_salesmen": {
            "cardinality": {
                "field": "salesman.id"
            }
        },

    }
}
```

:::note
cardinality aggregation produces approximate numbers.
:::

```SQL
GET /order/_search
{
    "size": 0,
    "aggs": {
        "values_count": {
            "value_count": {
                "field": "total_amount"
            }
        }

    }
}
```

:::note
So the `value_account` aggregation gives us the number of values that the aggregation used for doing its work.
:::

```SQL
GET /order/_search
{
    "size": 0,
    "aggs": {
        "stats": {
            "field": "total_amount"
        }
    }
}
```

:::note
The `stats` aggregation calculates the numbers returned by the min, max, sum, avg and value count aggregations.
:::

## Introduction to bucket aggregations

- Bucket aggregations create buckets of documents based on some criterion

```SQL
GET /order/_search
{
    "size": 0,
    "aggs": {
        "status_terms": {
            "terms": {
                "field": "status.keyword",
                "missing": "N/A",
                "min_doc_count": 0,
                "order": {
                    "_key": "asc"
                }
            }
        }
    }
}
```

## Document counts are approximate

- The reason why counts are not always accurate is because of the distributed nature of an Elasticsearch cluster.
  - Since an index is distributed across multiple shards, at least by default.
- The way the terms segregation works is that the coordinating note that is responsible for handling the search request prompts each shard for its top unique terms.
- The coordinating note then takes the results from each of the shards and uses those to compute the final result.

## Nested Aggregations

```SQL
GET /order/_search
{
    "size": 0,
    "query": {
        "range": {
            "total_amount": {
                "gte": 100
            }
        }
    }
    "aggs": {
        "status_terms": {
            "terms": {
                "field": "status.keyword",
            },
            "aggs": {
                "status_stats": {
                    "stats": {
                        "field": "total_amount"
                    }
                }
            }
        }
    }
}
```

:::note
So the terms aggregation runs in the context of the query.

While the stats aggregation runs in the context of its parent aggregation, being a bucket aggregation.
:::

:::note
Metric aggregations produce simple results and cannot contain sub-aggregations.

Bucket aggregations may contain sub aggregations which then operate on the buckets produced by the parent bucket aggregation.
:::

## Filtering out Documents

```SQL
GET /order/_search
{
    "size": 0,
    "aggs": {
        "low_value": {
            "filter": {
                "total_amount": {
                    "lt": 50
                }
            }
        },
        "aggs": {
            "avg_amount": {
                "avg": {
                    "field": "total_amount"
                }
            }
        }
    }
}
```

- The avg aggregation runs within the context of the filter, meaning that it only aggregates the documents that match the range query.
- The range query is a top-level aggregation, meaning that it runs in the context of the query, which is an implicit match_all query in this case, and the avg aggregation runs in the context of the filter aggregation, meaning that some documents have been filtered out.

## Defining bucket rules with filters

```
GET /recipe/_search
{
    "size": 0,
    "aggs": {
        "my_filter": {
            "filters": {
                "filters": {
                    "pasta": {
                        "match": {
                            "title": "pasta"
                        }
                    },
                    "spaghetti": {
                        "match": {
                            "title" "spaghetti"
                        }
                    }
                }
            },
            "aggs": {
                "avg_rating": {
                    "avg": {
                        "field": "ratings"
                    }
                }
            }
        }
    }
}
```

- This will take any documents whose title contains the term pasta and place them within a bucket named pasta. Likewise for Spaghetti.
- Thus, two buckets are created
- Also, calculated the average ratings for each of the buckets by using the avg aggregation.

## Range Aggregations

```SQL
GET /order/_search
{
    "size": 0,
    "aggs": {
        "amount_distribution": {
            "range": {
                "field": "total_amount",
                "ranges": [
                    {
                        "to": 50
                    },
                    {
                        "from": 50,
                        "to": 100
                    },
                    {
                        "from": 100
                    }
                ]
            }
        }
    }
}
```

```SQL
GET /order/_search
{
    "size": 0,
    "aggs": {
        "purchased_ranges": {
            "date_range": {
                "field": "purchased_at",
                "format": "yyyy-MM-dd",
                "keyed": true,
                "ranges": [
                    {
                        "from": "2016-01-01",
                        "to": "2016-01-01||+6M",
                        "key": "first_half"
                    },
                    {
                        "from": "2016-01-01||+6M",
                        "to": "2016-01-01||+1y",
                        "key": "second_half"
                    },
                ]
            },
            "aggs": {
                "bucket_stats": {
                    "stats": {
                        "field": "total_amount"
                    }
                }
            }
        }
    }
}
```

## Histograms

- A **histogram** dynamically builds buckets from a numeric field's value based on a specified interval.

```SQL
GET /order/_search
{
    "size": 0,
    "query": {
        "range": {
            "total_amount": {
                "gte": 10
            }
        }
    },
    "aggs": {
        "amount_distribution": {
            "histogram": {
                "field": "total_amount",
                "interval": 25,
                "min_doc_count": 1,
                "extended_bounds": {
                    "min": 0,
                    "max": 500
                }
            }
        }
    }
}
```

- Date Histogram example:

```SQL
GET /order/_search
{
    "size": 0,
    "aggs": {
        "orders_over_time": {
            "date_histogram": {
                "field": "purchased_at",
                "interval": "month",
            }
        }
    }
}
```

## Global Aggregations

```SQL
GET /order/_search
{
    "query": {
        "range": {
            "total_amount": {
                "gte": 100
            }
        }
    },
    "size": 0,
    "aggs": {
        "all_orders": {
            "global": {},
            "aggs": {
                "stats_amount":{
                    "stats": {
                        "field": "total_amount"
                    }
                }
            }
        }
    }
}
```

- The global aggregation is not influenced by the search query.
- It uses the context of the index and type that the query is run against and uses all of the documents that are available within this context.

## Missing Field Values

```SQL
GET /order/_search
{
    "size": 0,
    "aggs": {
        "orders_without_status": {
            "missing": {
                "field": "status.keyword"
            }
        }
    }
}
```

- Since the missing aggregation is a bucket aggregation, it creates a bucket with the orders that either don't have a status field at all or have a value of null.

## Aggregating Nested Objects

```SQL
GET /department/_search
{
    "size": 0,
    "aggs": {
        "employees": {
            "nested": {
                "path": "employees"
            }
        }
    }
}
```
