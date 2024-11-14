---
sidebar_position: 5
---

# Controlling Query Results

## Specifying the result format

```SQL
GET /recipes/_search?format=yaml
{
    "query": {
        "match": { "title": "pasta" }
    }
}
```

- Make the response pretty

```SQL
GET /recipes/_search?pretty
{
    "query": {
        "match": { "title": "pasta" }
    }
}
```

## Source Filtering

```SQL
GET /recipes/_search
{
    "_source": "created",
    "query": {
        "match": { "title": "pasta" }
    }
}
```

- Example returns match all properties within the objects stored within the ingredients array

```SQL
GET /recipes/_search
{
    "_source": "ingredients.*",
    "query": {
        "match": { "title": "pasta" }
    }
}
```

- Example that name property is no longer returned for each object within the ingredients field

```SQL
GET /recipes/_search
{
    "_source": {
        "includes": "ingredients.*",
        "excludes": "ingredients.name"
    }
    "query": {
        "match": { "title": "pasta" }
    }
}
```

## Specifying the result size

- Two Examples below output same thing where 2 matches are shown

```SQL
GET /recipes/_search?size=2
{
    "_source": false,
    "query": {
        "match": {
            "title": "pasta"
        }
    }
}

GET /recipes/_search
{
    "size": 2,
    "_source": false,
    "query": {
        "match": {
            "title": "pasta"
        }
    }
}
```

## Specifying an offset

```SQL
GET /recipes/_search
{
    "size": 2,
    "_source": false,
    "from": 4,
    "query": {
        "match": {
            "title": "pasta"
        }
    }
}
```

:::note
The `from` paramter is like the offset clause in SQL
:::

## Pagination

- [Pagination Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-body.html#request-body-search-search-after)
- `total_pages = ceil(total_hits / page_size)`
- `from = (page_size * (page_number - 1))`

## Sorting Results

```SQL
GET /recipes/_search
{
    "_source": false,
    "query": {
        "match_all": {}
    },
    "sort": [
        "preparation_time_minutes"
    ]
}

GET /recipes/_search
{
    "_source": "created",
    "query": {
        "match_all": {}
    },
    "sort": [
        { "created": "desc" }
    ]
}
```

## Sorting by multi-value fields

```SQL
GET /recipes/_search
{
    "_source": "ratings",
    "query": {
        "match_all": {}
    },
    "sort": [
        {
            "ratings": {
                "order": "desc",
                "mode": "avg"
            }
        }
    ]
}
```
