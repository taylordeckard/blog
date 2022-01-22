---
author: Taylor Deckard
title: Elasticsearch Intro
date: 2021-01-21
description: Getting started with Elasticsearch
---

I've started working on a proof-of-concept to improve query performance of a large dataset (5M+ rows.) The data is currently stored in a MySQL database.

The service is required to search, sort, filter, and paginate the data. Nowadays, these requirements are standard practice. However, with such a large dataset, some of the database queries are taking > 3 seconds, even with table partitioning.

My theory is that Elasticsearch will perform better than RDBMS for this use case. Only one way to find out...

![Time to experiment!](/blog/images/elasticsearch-intro/dog_experiment.gif)

## Running Elasticsearch Locally

I already have [Docker](https://www.docker.com/products/docker-desktop) and [docker-compose](https://docs.docker.com/compose/install/) installed, so I just need to create a docker-compose config.

```yaml
version: "3.9"
# Run 3 Elasticsearch containers simultaneously (es01, es02, es03)
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.16.3
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    # Allow unlimited memory to be locked by this container process
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      # Map data01 directory to container (for persistence)
      - data01:/usr/share/elasticsearch/data
    ports:
      # Map local port 9200 to container port 9200
      - 9200:9200
    networks:
      - elastic
  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.16.3
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data02:/usr/share/elasticsearch/data
    networks:
      - elastic
  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.16.3
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data03:/usr/share/elasticsearch/data
    networks:
      - elastic

volumes:
  data01:
    driver: local
  data02:
    driver: local
  data03:
    driver: local

# ES Nodes run on a shared network that is bridged to the local machine
networks:
  elastic:
    driver: bridge
```

Elasticsearch should be ready to go. Before running though, I'll need to bump up Docker's memory resources to > 4GB.
![Docker Resources](/blog/images/elasticsearch-intro/docker_resources.png)

Start docker-compose with
```bash
docker-compose up
```

## Creating an Index
Now that Elasticsearch is running, it's time to learn how to use it: [REST API docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/rest-apis.html).

First, I want to create an _index_ (table in SQL) for my dataset so I can add _documents_ (rows) to it.

I create an index called `assets` ([API Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html)):
```bash
curl -X PUT 'http://localhost:9200/assets'
```
The following is shown in the docker-compose logs:
```json
{"type": "server", "timestamp": "2022-01-21T15:50:11,935Z", "level": "INFO", "component": "o.e.c.m.MetadataCreateIndexService", "cluster.name": "es-docker-cluster", "node.name": "es02", "message": "[assets] creating index, cause [api], templates [], shards [1]/[1]", "cluster.uuid": "6BFPOn84Q8qFIfS2FSmsBw", "node.id": "Sm9PsrFzTu6tkK0tAt0qvw"  }
```
which indicates the index creation was successful.

Another way to check that the index creation was successful is to do
```bash
curl --head 'http://localhost:9200/assets'
```
A 200 response means that the index exists, whereas 404 means it doesn't.

## Working with Documents

Adding a document to the index is simple ([API Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html)):
```bash
curl -X POST  'http://localhost:9200/assets/_doc/' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Test Data"
  }'
```
Then retrieve the document with ([API Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-search.html))
```bash
curl -X GET 'localhost:9200/assets/_search?pretty' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "match":  {
        "name": "Test Data"
      }
    }
  }'
```

## Mocking Data

The next step is to populate the local Elasticsearch database up with mock documents to mimic a production environment. I'm familiar with Node.js, so I'll write a quick script to do this.

Create a directory to contain the script.
```bash
mkdir mock-data && cd mock-data
```

Add a package.json
```bash
npm init -y
```

Now, I'll add a couple of dependencies. To create mock-data I'm using a fork of faker.js, ([community-faker](https://www.npmjs.com/package/community-faker).) I'll also use [node-fetch](https://www.npmjs.com/package/node-fetch) to make HTTP requests.

```bash
# shorthand for npm install, -S writes the dependencies to package.json
npm i -S community-faker node-fetch
```

Next, write the script: index.js. (**Note:** I'm using Node.js v14.16.1)
```javascript
import faker from 'community-faker';
import fetch from 'node-fetch';

// Number of documents to insert
const NUM_DOCUMENTS = 25000;
const ES_HOST = 'http://localhost:9200';
// The ES index to insert records into
const INDEX = 'assets';

(async function run () {
  for (let i = 0; i < NUM_DOCUMENTS; i++) {
    const doc = {
      name: faker.name.findName(),
      id: faker.datatype.uuid(),
    };
    // Send document to ES via http API
    const response = await fetch(`${ES_HOST}/${INDEX}/_doc`, {
      // node-fetch requires stringifying the json body
      body: JSON.stringify(doc),
      headers: {
        // Content-Type header is required for ES APIs with bodies
        'Content-Type': 'application/json',
      },
      method: 'post',
    })

    if (!response.ok) {
      // If an error occurred, log the error and exit
      console.log(await response.json());
      break;
    }
    // Write out progress on a single line
    process.stdout.clearLine();
    process.stdout.cursorTo(0);
    process.stdout.write(`${i + 1} / ${ NUM_DOCUMENTS } documents inserted`);
  }
})();
```

Finally, run the script.
```bash
node index.js
```

When finished, confirm the data has been ingested.
```bash
curl -X GET 'localhost:9200/assets/_search?pretty' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "match_all": {}
    },
    "size": 10,
    "from": 0
  }'
```

## Evaluating Response Time
With 25k documents, queries are speedy at around 10ms. I noticed a problem though when trying to query for the last page:
```json
{
  "error": {
    "root_cause": [
      {
        "type": "illegal_argument_exception",
        "reason": "Result window is too large, from + size must be less than or equal to: [10000] but was [20010]. See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the [index.max_result_window] index level setting."
      }
    ],
  ...
}
```
This is a nice descriptive error message. The default maximum documents allowed for from/size pagination is 10k and an error was thrown when I tried to start the query at 20k. I could of course, raise the max_result_window setting to something higher, but this is inefficient. The error message recommends using the scroll api, documented [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/paginate-search-results.html#scroll-search-results). However...
> We no longer recommend using the scroll API for deep pagination. If you need to preserve the index state while paging through more than 10,000 hits, use the `search_after` parameter with a point in time (PIT).

So what is [`search_after` with PIT](https://www.elastic.co/guide/en/elasticsearch/reference/current/paginate-search-results.html#search-after)? It's actually pretty simple.

Before making calls that require deep pagination, first request a Point in Time ID from the index.
```bash
curl -X POST 'localhost:9200/assets/_pit?keep_alive=1m'
```
This will return a json response that looks like this:
```json
{"id":"z4S1AwEGYXNzZXRzFlRLaVNVSFpyVHMtNS1hZ1ZKS1pXNWcAFnREZjlWcFBxVFN5UkJuTk1XQ0tPOVEAAAAAAAAAAJ4WSU5tZ21Xc2NTYy03OHVwUUg4Z2pBQQABFlRLaVNVSFpyVHMtNS1hZ1ZKS1pXNWcAAA=="}
```
The PIT `id` above can be used in a search query like this:
```bash
curl -X GET 'localhost:9200/_search' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "match": {
        "name": "Test Data"
      }
    },
    "sort": [{ "name.keyword": "desc" }],
    "size": 10,
    "pit": {
      "id":"z4S1AwEGYXNzZXRzFlRLaVNVSFpyVHMtNS1hZ1ZKS1pXNWcAFnREZjlWcFBxVFN5UkJuTk1XQ0tPOVEAAAAAAAAAAJ4WSU5tZ21Xc2NTYy03OHVwUUg4Z2pBQQABFlRLaVNVSFpyVHMtNS1hZ1ZKS1pXNWcAAA==",
      "keep_alive": "1m"
    }
  }'
```
In response, I will receive documents 1-10 along with a `sort` field.
```json
{
  ...
  "sort": [
    "fffbfc0",
    25358
  ]
}
```

To query for documents 11-20, I'll set the `search_after` parameter in my next query to the `sort` value, like this:

```bash
curl -X GET 'localhost:9200/_search' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "match": {
        "name": "Test Data"
      }
    },
    "sort": [{ "name.keyword": "desc" }],
    "size": 10,
    "pit": {
      "id":"z4S1AwEGYXNzZXRzFlRLaVNVSFpyVHMtNS1hZ1ZKS1pXNWcAFnREZjlWcFBxVFN5UkJuTk1XQ0tPOVEAAAAAAAAAAJ4WSU5tZ21Xc2NTYy03OHVwUUg4Z2pBQQABFlRLaVNVSFpyVHMtNS1hZ1ZKS1pXNWcAAA==",
      "keep_alive": "1m"
    },
    "search_after": [
      "fffbfc0",
      25358
    ]
  }'
```

A downside to this approach is that querying the last page is not as simple as with the `from` parameter. An arbitrary page cannot be selected. Instead, a starting page must first be retreived. This will work fine for pagination with simple "Previous" and "Next" buttons. It does not work, however, for pagination that allows the user to select a specific page.

As a compromise, the UI could offer Previous/Next pagination until total query results are reduced below 10k, then offer the option to select a specific page. This could be suitable for cases when the need for deep pagination is present but uncommon.

## Finishing Up for the Day
There is still more investigation needed. Analyzing performance on 25k documents is a good start, but I will need to increase that number quite a bit to get a realistic estimate of improvement over the current system. Also, I've barely scratched the surface of Elasticsearch's capabilities. The research continues...
