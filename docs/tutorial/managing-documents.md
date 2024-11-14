---
sidebar_position: 1
---

# Managing Documents

The following sections will use the Kibana console tool on the localhost url.

## Creating & Deleting Indices

- Deleting an index:
- **Deleting pages index example**

```
DELETE /pages
```

- Creating an index:
- **Adding products index example**
  - To specify index settings, add a request body
  - Add a JSON object
  - First line specifies the HTTP verb and endpoint
  - Following lines specify the JSON request body

```SQL
PUT /products
{
    "settings": {
        "number_of_shards": 2,
        "number_of_replicas": 2,
    }
}
```

## Indexing Documents

- Index a document using POST request
- **products index example**:

```SQL
POST /products/_doc
{
    "name": "Coffee Maker",
    "price": 64,
    "in_stock": 10
}

// set id to 100
PUT /products/_doc/100
{
    "name": "Toaster",
    "price": 49,
    "in_stock": 4
}
```

## Retrieving documents by ID

- Retrieve Toaster example at id 100

```SQL
GET /products/_doc/100
```

## Updating documents

- Updating Toaster example at id 100 to have less count down to 3 in stock

```SQL
POST /products/_update/100
{
    "doc": {
        "in_stock": 3
    }
}
```

- Adding New Fields
- Adding electronics field to Toaster Example

```SQL
POST /products/_update/100
{
    "doc": {
        "tags": ["electronics"]
    }
}
```

### Documents are immutable

:::info

- Elasticsearch documents are immutable
- Instead, we replaced documents entirely
- The Update API is simpler and **saves network traffic**
  - The current document is retrieved
  - The field values are changed
  - The existing document is replaced with the modified document
  - Replacing is less requests compared to modifying, which is good.

:::

## Scripted updates

- Elasticsearch supports scripting, allowing you to write custom logic or code while assessing a document's values
- **Scripting to decrease count for Toaster example**:
  - `ctx` is a variable, short word for context
  - `_source` property gives an object containing the document's fields
  - `in_stock` field is the custom count of items created earlier
  - `--` decrements and `++` increments

```SQL
POST /products/_update/100
{
    "script": {
        "source": "ctx._source.in_stock--"
    }
}
```

- **Scripting to assign a new count for Toaster example**:

```SQL
POST /products/_update/100
{
    "script": {
        "source": "ctx._source.in_stock = 10"
    }
}
```

- Could pass in parameters
- **Scripting to account for parameters making changes to count for Toaster example**:

```SQL
POST /products/_update/100
{
    "script": {
        "source": "ctx._source.in_stock -= params.quantity"
        "params": {
            "quantity": 4
        }
    }
}
```

- **Scripting to decrease count for Toaster, accounting for edge cases, example**:
  - add an if clause to check if stock is zero. Cannot decrement when 0.
  - if the if clause is met, then no changes are applied to the document and the "result" key is set to `noop`

```SQL
POST /products/_update/100
{
    "script": {
        "source": """
            if(ctx._source.in_stock == 0){
                ctx.op = 'noop';
            }

            ctx._source.in_stock--;
        """
    }
}
```

- Similarly, could change to this logic that does the same thing:

```SQL
POST /products/_update/100
{
    "script": {
        "source": """
            if(ctx._source.in_stock > 0){
                ctx._source.in_stock--;
            }
        """
    }
}
```

:::tip
NOTE: The different with this code compared to the one before is that it will always yield a result of "updated", regardless of whether or not the field value was actually changed.

If it is important to detect if nothing was changed go with the new logic.
:::

## Upserts

- **Upserting**: to conditionally update or insert a document based on whether or not the document already exists.

  - In other words, if the document already exists, a script is ran, and if it doesn't, a new document is indexed.

- **Upserting Example, adding a new product**:
  - Since there is no document with id 101, the contents of the upsert option is indexed as a new document.
  - To verify in the console, when you run the query, the "result" key contains a value `created`.
  - If you run the query again, the "result" key contains a value `updated`.
    - The `in_stock` value will increase to `6`.

```SQL
POST /products/_update/101
{
    "script": {
        "source": "ctx._source.in_stock++"
    },
    "upsert": {
        "name": "Blender",
        "price": 399,
        "in_stock": 5
    }
}
```

## Replacing Documents

- **Toaster Example**:
  - Previous Toaster had a price of 49 and additional tag fields.
  - Replace the document to show a Toaster with a price of 79.
    - The query will also give a document without the additional tag fields.

```SQL
PUT /products/_doc/100
{
    "name": "Toaster",
    "price": 79,
    "in_stock": 4
}
```

## Deleting Documents

- Delete document at id 101

```SQL
DELETE /products/_doc/101
```

## Understanding Routing

- Routing is used to let Elasticsearch know where to store documents and how to find documents once they have been indexed.
- **Routing** is the process of resolving a shard for a document.
- Routing is 100% transparent when using Elasticsearch.
- It is possible to customize routing for various purposes.
- The default routing strategy ensures that documents are distributed evenly.

FORMULA:
`shard_num = hash(_routing) % num_primary_shards`

:::warning

- If you manipulate the `num_primary_shards`, Elasticsearch may not be able to locate documents even though it exists.
- Modifying the number of shards requires creating a new index and reindexing documents into it.

:::

## How Elasticsearch reads data

- A read request is received and handled by a **coordinating node**.
- Routing is used to resolve the document's replication group.
- Elasticsearch uses a technique called Adaptive Replica Selection (ARS).
  - If Elasticsearch retrieved the document directly from the primary shard, all retrievals would end up on the same shard, which does not scale well.
  - Instead, a shard is chosen from the replication group.
  - ARS selects the best performance shard copy.
  - Then, the coordinating node sends the read request to that shard.
  - When the shard responds, the coordinating node collects the response and sends it to the client.
- ARS helps reduce query response times.
- ARS is an intelligent load balancer.

## How Elasticsearch writes data

- Write operations are sent to primary shards
  - Primary shards validate the structure of the request and its field values.
- The primary shard forwards the operation to its replica shards in parallel to improve performance.
- Primary terms and sequence numbers are used to recover from failures
  - **Primary terms** are a way to distinguish old and new primary shards.
  - A primary term is a counter for how many times the primary shard has changed.
  - The primary term is appended to write operations.
  - A **sequence number** is a counter that is incremented for each write operation until the primary shard changes.
  - Sequence number is appended to write operations together with the primary term.
  - The primary shard increases the sequence number.
  - Sequence numbers enable Elasticsearch to order write operations.

### Recovering when a primary shard fails

- Primary terms and sequence numbers are key when Elasticsearch needs to recover from a primary shard failure.
  - Enables Elasticsearch to more efficiently figure out which write operations need to be applied
- For large indices, this process is really expensive.
- Global and local checkpoints help speed up the recovery process
  - **Checkpoints** are sequence numbers.
  - Each replication group has a global checkpoint
  - Each replica shard has a local checkpoint
- **Global checkpoints** is the sequence number that all active shards within a replication group have been aligned at least up to
  - This means any operations containing a sequence number lower than the global checkpoint have already been performed on all shards within the replication group.
  - If a primary shard fails and rejoins the cluster at a later point, Elasticsearch only needs to compare the operations that are above the global checkpoint that it last knew about.
- **Local checkpoints** is the sequence number for the last write operation that was performed.
- Primary terms and sequence numbers are available within responses.

## Understanding document versioning

- Versioning is not a revision history of documents.
- Elasticsearch stores an `_version` metadata field with every document
  - The value is an integer
  - When the document is updated, the value will increment by 1.
  - The `_version` field is returned when retrieving documents

### Types of versioning

- Default type of versioning is called **internal versioning**.
- **External Versioning**: used when versions are maintained outside of Elasticsearch.
  - e.g. when documents are also stored in a database
    ```SQL
    PUT /products/_doc/123?version=521&version_type=external
    {
        "name": "Coffee Maker",
        "price": 64,
        "in_stock": 10
    }
    ```

### Purpose of Versioning

- You can see how many times a document has been modified.
  - Probably not useful
- This versioning is hardly used.

## Optimistic Concurrency Control

- Prevent overwriting documents due to concurrent operations
  - If Primary term and sequence term matches what's currently there, then success response, else error response.
  - Elasticserach will reject a write operation if it contains the wrong primary term or sequence number

```SQL
POST /products/_update/100?if_primary_term=X&if_seq_no=X
{
    "doc": {
        "in_stock": 123
    }
}
```

### How to handle failures?

- Handle the situation at the application level
  - Retrieve the document again
- Use `_primary_term` and `_seq_no` for a new update request
- Remember to perform any calculations that use field values again

## Update by Query

- `"conflicts": "proceed"` will prevent the query from being aborted when there is a version conflict.

```SQL
POST /products/_update_by_query
{
  "conflicts": "proceed",
  "script": {
    "source": "ctx._source.in_stock--"
  },
  "query": {
    "match_all": {}
  }
}
```

- The query creates a snapshot to do optimistic concurrency control
  - Prevents overwriting changes made after the snapshot was taken.
  - The query may take a while to finish if updating many documents.
- Each document's primary term and sequence number is used.
  - A document is only updated if the values match the ones from the snapshot (optimistic concurrency control)
- Number of version conflicts is returned with the `version_conflicts` key.
- Search queries and bulk requests are sent to replication groups sequentially
  - Elasticsearch retries these queries up to 10 times
  - If the queries still fail, the whole query is aborted
    - Any changes already made to the document are not rolled back
- The API returns information about failures
- If a document had been modified since taking the snapshot, the query is aborted.
  - This is checked with the document's primary term and sequence number
- To count version conflicts instead of aborting the query, the `conflicts` option can be set to `proceed`.

## Delete by Query

- Delete multiple documents within a single query
- **Example: Delete all documents within the "products" index**:
  - Query will delete all documents that match the query
  - `match_all` is empty hence it deletes all document

```SQL
POST /products/_delete_by_query
{
  "query": {
    "match_all": {}
  }
}
```

## Batch Processing

### Intro to Bulk API

- Index, Update, and Delete many documents at once at a large scale.
  - Achieved by processing individual requests in batches using Bulk API.
- The [Bulk API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html) expects data formatted using the NDJSON specification

```
action_and_metadata\n
optional_source\n
action_and_metadata\n
optional_source\n
```

- In the Kibana console, let's index a new document
  - `"_id"` is optional

```SQL
POST /_bulk
{ "index": { "_index": "products", "_id": 200} }
{ "name": "Expresso Machine", "price": 199,, "in_stock": 5 }
{ "create": { "_index": "products", "_id": 201 } }
{ "name": "Milk Frother", "price": 149, "in_stock": 14 }
```

:::note

The code above will create two new products

:::

```SQL
POST /products/_bulk
{ "update": { "_id": 201 } }
{ "doc": { "price": 129 } }
{ "delete": { "_id": 200 } }
```

:::note

The code above doesn't need index since it's specified after `POST`.
It will update document with id 201 and delete document with id 200.

:::

### Things to be aware of

:::warning

- The HTTP `Content-Type` header should be set to

```
Content-Type: application/x-ndjson
```

Using HTTP clients, we need to handle this ourselves.

:::

:::warning

- Each line must end with a newline character such as `\n` or `\r\n`
  - This includes the last line
  - In a text editor, just leave the last line empty. Don't type `\n` or `\r\n`.

:::

:::warning

- A failed action will not affect other actions
  - Neither will the bulk request as a whole be aborted
- The Bulk API returns detailed info about each action
  - Inspect the items key to see if a given action succeeded
    - The order is the same as the actions within the request
  - The `errors` key conveniently tells us if any errors occurred.

:::

### When to use the Bulk API

- Needed when performing lots of write operations at the same time
- The Bulk API is more efficient than sending individual write requests
  - A lot of network round trips are avoided

### Final note

- Routing is used to resolve a document's shard
  - Routing could be customized
- The Bulk API supports optimistic concurrency control
  - Include the `if_primary_term` and `if_seq_no` parameters within the action metadata

## Importing data with cURL

- [Download products-bulk.json](../assets/managing-documents-assets/products-bulk.json)
- In terminal, the working directory is the file location of your test file: `products-bulk.json`. Then, enter:

```
curl -H "Content-Type: application/x-ndjson" -XPOST http://localhost:9200/products/_bulk --data-binary "@products-bulk.json"
```
