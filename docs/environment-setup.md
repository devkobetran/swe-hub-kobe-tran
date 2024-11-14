---
sidebar_position: 2
---

# Environment Setup

## Local Setup

- Docker works but complicated
- Elasticsearch available on MacOS
- Opensearch not available on MacOS

## Managed Services

- Elasticsearch:

  - Elastic Cloud
  - Google Cloud
  - Microsoft Azure

- OpenSearch:
  - Bonsai (Free forever)
  - AWS
  - DigitalOcean
  - Aiven
  - Instacluster
  - Exoscale

## Hosting OpenSearch on Bonsai

[Bonsai](https://bonsai.io/)

## Hosting Elasticsearch & Kibana on Elastic Cloud

- The instructions below are purely for MacOS

### Download

[Download Elasticsearch](https://www.elastic.co/downloads/elasticsearch)

[Download Kibana](https://www.elastic.co/downloads/kibana)

### Setting up Elasticsearch

- Go to folder where elasticsearch is located in terminal.
- Disable Gatekeeper for the elasticsearch directory: `xattr -d -r com.apple.quarantine elasticsearch`
- `cd elasticsearch`
- `bin/elasticsearch` to invoke a script named elasticsearch within the bin directory
- Make sure the passwords and tokens are displayed.

:::warning
Things may keep running in the terminal ... so pay attention and scroll to the summary with the password and tokens.
:::

:::info

- Resetting the elastic user's password: `bin/elasticsearch-reset-password -u elastic`
- Generate a new Kibana enrollment token: `bin/elasticsearch-create-enrollment-token --scope kibana`
  - Valid for 30 minutes
    :::

### Setting up Kibana

- Go to folder where kibana is located in terminal.
- Disable Gatekeeper for the kibana directory: `xattr -d -r com.apple.quarantine kibana`
- `cd kibana`
- `bin/kibana`
- Copy the generated localhost url and paste it in the browser.
  - It asks for an Enrollment token.
  - Grab the enrollment token from Elasticsearch and paste it there. Then, click **Configure Elastic**.
  - The Elastic login will appear. Enter the elastic superuser credentials generated from the terminal for elasticsearch.

## Inspecting the cluster

### Using the Console tool in Kibana

- After logging in to the elastic login page through the localhost url when you ran kibana, a dashboard will appear.
- Expand the menu bar in the top left corner.
  - Under **Management**, select **Dev Tools**
- Modify the console, with your own requests such as:

  - Retrieving the cluster's health:

  ```
  // _cluster is the API
  // health is the command
  GET /_cluster/health
  ```

  - Listing "nodes"

  ```
  // _cat is the CAT API
  // nodes is the command
  // v is a query parameter which adds a descriptive header row to the output
  GET /_cat/nodes?v
  ```

  - Creating a new index called "pages"

  ```
  PUT /pages
  ```

  - List all the shards within the cluster
    - It will detail which index the shard will belong to

  ```
  GET /_cat/shards?v
  ```

### Resources

- [cat nodes API](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-nodes.html)
- [Nodes info API](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-nodes-info.html)

## Sending queries with cURL

### Installation of cURL

:::note
This step is probably not needed for macOS systems.
:::

- [Install cURL, if necessary](https://curl.se/download.html)

### Steps

- Open terminal and be in `elasticsearch` directory.
- `curl --cacert config/certs/http_ca.crt -u elastic -X GET https://localhost:9200`
  - Password prompt will appear. Enter superuser password for local login.
- Insecure way and revealing password:

```
curl --cacert config/certs/http_ca.crt -u elastic:YOUR_PASSWORD -X GET https://localhost:9200
```

- Use Search API for a products index.

```
curl --cacert config/certs/http_ca.crt -u elastic:YOUR_PASSWORD -X GET -H "Content-Type:application/json" https://localhost:9200/products/_search -d '{ "query": { "match_all": {} } }'
```
