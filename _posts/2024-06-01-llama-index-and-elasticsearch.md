---
title: "Using LlamaIndex with Elasticsearch"
author: imotov
pin: false
categories: [how-tos, intermediate]
tags: [elasticsearch, llama-index, how-to]
---

## Introduction

In the world of search engines and data indexing, two powerful tools have emerged: Elasticsearch and LlamaIndex. This blog post will introduce both, showcase how to integrate them, and demonstrate practical examples. We will also explore how to optimize your Elasticsearch index to reduce its size effectively.

## What is LlamaIndex?

LlamaIndex is a powerful library designed for building context-augmented systems based on large language models (LLMs). It supports use cases such as Retrieval-Augmented Generation (RAG) systems, Document Understanding and Extraction, and Autonomous Agents. LlamaIndex is compatible with multiple vector stores as backends, including Elasticsearch.

## What is Elasticsearch?

Elasticsearch is a widely-used, open-source search and analytics engine known for its speed, scalability, and flexibility. Built on Apache Lucene, Elasticsearch provides distributed, full-text search capabilities and is commonly used for log and event data analysis, as well as powering search features in various applications. Crucially for our use case, Elasticsearch also functions as a vector database, making it ideal for handling complex data structures combined with semantic searches.

## Initial Setup

To follow along with the examples below, you will need to install the following packages:

```plaintext
llama-index
llama-index-vector-stores-elasticsearch
py7zr
xmltodict
wikitextparser
Python-dotenv
```

You can run the following code snippets in a Jupyter notebook or as a standalone Python file. You can find the complete jupyter notebook at <https://github.com/akaula/blog_notebooks/blob/main/blog_notebooks/llama_index_and_elasticsearch_p1.ipynb>.

In addition to installing the necessary packages, we will need to set up a few environment variables. You can specify these variables either on the command line or in a `.env` file.

If you are running elasticsearch on premises you will need:

```bash
OPENAI_API_KEY=<your OpenAI key>
ES_URL=<Elasticsearch url>
ES_USER=<Elasticsearch user>
ES_PASSWORD=<Elasticsearch password>
```

If you are using an Elastic Cloud instance:

```bash
OPENAI_API_KEY=<your OpenAI key>
ES_CLOUD_ID=<Elastic Cloud ID>
ES_API_KEY=<Elastic API Key>
```

## Creating a LLamaIndex Vector Store Backed by Elasticsearch

Now we need to instantiate an Elasticsearch-backed LlamaIndex vector store. This is done by creating an `ElasticsearchStore` and then building a vector store on top of it.

```python
# Setup elasticsearch and LlamaIndex
import os
from dotenv import load_dotenv

from llama_index.core import VectorStoreIndex
from llama_index.vector_stores.elasticsearch import ElasticsearchStore

# Load .env file
load_dotenv()
ES_URL = os.getenv("ES_URL")
ES_USER = os.getenv("ES_USER")
ES_PASSWORD = os.getenv("ES_PASSWORD")
ES_CLOUD_ID = os.getenv("ES_CLOUD_ID")
ES_API_KEY = os.getenv("ES_API_KEY")
ES_INDEX_NAME = os.getenv("ES_INDEX_NAME", "matrixfilms")

# Create LlamaIndex vector store
vector_store = ElasticsearchStore(
    index_name=ES_INDEX_NAME,
    es_url=ES_URL,
    es_user=ES_USER,
    es_password=ES_PASSWORD,
    es_cloud_id=ES_CLOUD_ID,
    es_api_key=ES_API_KEY,
)

# Create index from the store
index = VectorStoreIndex.from_vector_store(vector_store=vector_store)

```

## Loading Data

To make our exploration more engaging, we’ll utilize data from The Matrix Fandom Wiki. This dataset includes detailed descriptions of characters, events, and technologies from The Matrix universe, enabling us to build a system with expert knowledge of The Matrix. The following snippet downloads the wikipedia dump and unzips it into the current directory

```python
# Download the test data

from pathlib import Path
from py7zr import SevenZipFile
import requests

def download_wiki(dump_file):
    dump_url = f"https://s3.amazonaws.com/wikia_xml_dumps/{dump_file[:1]}/{dump_file[:2]}/{dump_file}.7z"
    dump_7z_file_path = f"{dump_file}.7z"
    response = requests.get(dump_url)
    if response.status_code == 200:
        with open(dump_7z_file_path, "wb") as file:
            file.write(response.content)
        with SevenZipFile(dump_7z_file_path, mode="r") as archive:
            archive.extractall(path=".")
    else:
        raise RuntimeError(f"Failed to download the file. HTTP Status Code: {response.status_code}")

dump_file = "matrixfilms_pages_current.xml"

if not Path(f"{dump_file}").exists():
    download_wiki(dump_file)
```

Now we need to extract text from all non-special pages:

```python
# Parse wiki documents

import wikitextparser as wtp
import xmltodict

def parse_wiki_xml(file_path, limit = None):
    docs = []
    namespaces = []
    page = 0

    def process_page(title, text):
        nonlocal docs
        doc = {"content": text, "meta": {"title": title, "id": page}}
        docs.append(doc)

    def handle_content(address, content):
        nonlocal page, namespaces
        name = address[1][0]
        if name == "siteinfo":
            # We collect a set of namespaces that indicate special purpose wiki pages that we will ignore
            for namespace_elem in content["namespaces"]["namespace"]:
                namespace = namespace_elem.get("#text")
                if namespace:
                    namespaces.append(namespace)
        elif name == "page":
            title = content["title"]
            # Ignore special pages
            if any(title.startswith(namespace + ":") for namespace in namespaces):
                return True
            revision = content.get("revision")
            if revision:
                text = revision.get("text").get("#text")
                # Use wikitextparser to extract the plain text of the page
                text = wtp.parse(text).plain_text()
                process_page(title, text)
                page = page + 1
        return not limit or page < limit

    with open(file_path, "r", encoding="utf-8") as f:
        try:
            xmltodict.parse(f.read(), item_depth=2, item_callback=handle_content)
        except xmltodict.ParsingInterrupted:
            print("ParsingInterrupted... stopping...")
    return docs

docs = parse_wiki_xml(dump_file)

```

Once we have the pages indexing these pages into elasticsearch is as easy as the following snippet. This process takes about 10 minutes and will cost you about $0.07 if you run it against OpenAI API.

```python
# Index documents in elasticsearch
from llama_index.core import Document

# That takes about 10 minutes and will cost you about $0.07
for doc in docs:
    index.insert(Document(text=doc["content"], doc_id=doc["meta"]["id"], extra_info=doc["meta"]))

```

## Querying Data

Now we can make a query engine out of it and start asking some questions

```python
# Now we can perform searches
import textwrap
query_engine = index.as_query_engine()

print(textwrap.fill(query_engine.query("Tell me about battle of Zion").response, width=80))
```

Your answer might be slightly different, but you should expect to see something like this:

```plain
The Battle of Zion was a pivotal event during the First Machine war, where the
Machines launched a massive assault on the human city of Zion. The battle
involved intense fighting between the human defenders and the Machine forces,
including Sentinels and Diggers. Despite initial setbacks, the humans managed to
activate an EMP that temporarily halted the Machines' advance. Ultimately, a
peace treaty was brokered between Neo and the Machines, leading to the end of
the war and the preservation of both Zion and the Matrix.
```

Congratulations! You have built your own RAG system on top of Elasticsearch. But we are not done yet. The default mapping that LlamaIndex is using is very inefficient and might make your index quite bloated with a large amount of data. So, let’s see if we can improve it.

## Improving Elasticsearch Storage

Let’s start by checking the size of the existing index by switching to Kibana and running the following command:

```plaintext
GET matrixfilms/_stats/store?human&filter_path=indices.*.primaries.store.size
```

It should return us something like this:

```json
{
  "indices": {
    "matrixfilms": {
      "primaries": {
        "store": {
          "size": "71.8mb"
        }
      }
    }
  }
}
```

Next, let’s see how this space is used by running the following command:

```plaintext
POST matrixfilms/_disk_usage?run_expensive_tasks=true&filter_path=*.fields.*.total
```

This will return a list of fields and their sizes:

```json
{
  "matrixfilms": {
    "fields": {
      "_id": {
        "total": "129.4kb"
      },
      "_ignored": {
        "total": "97.5kb"
      },
      "_primary_term": {
        "total": "0b"
      },
      "_seq_no": {
        "total": "10.2kb"
      },
      "_source": {
        "total": "56.5mb"
      },
      "_version": {
        "total": "0b"
      },
      "content": {
        "total": "1mb"
      },
      "content.keyword": {
        "total": "112.9kb"
      },
      "embedding": {
        "total": "13.3mb"
      },
      "metadata._node_content": {
        "total": "438.9kb"
      },
      "metadata._node_type": {
        "total": "340b"
      },
      "metadata._node_type.keyword": {
        "total": "216b"
      },
      "metadata.doc_id": {
        "total": "14.5kb"
      },
      "metadata.document_id": {
        "total": "14.5kb"
      },
      "metadata.id": {
        "total": "10kb"
      },
      "metadata.ref_doc_id": {
        "total": "14.5kb"
      },
      "metadata.title": {
        "total": "31.7kb"
      },
      "metadata.title.keyword": {
        "total": "46.9kb"
      }
    }
  }
}
```

This list shows us that the majority of space is used by \_source, which makes sense. We are indexing embeddings that are represented as 1536 floats in the source on top of the text of each page. But, unless we are planning to keep re-indexing this index, we don’t need to store them in the \_source. We can also notice that dynamic mapping has created a whole bunch of unnecessary fields such as:

- `metadata._node_content`: we are not going to search this field
- `metadata._node_type` and `metadata._node_type.keyword`: we are unlikely to search this field either, and we definitely don’t need two versions of it
- `metadata.title.keyword`: that is also not needed.

After cleanup, we can create the following mapping and reindex our index:

```plaintext
PUT matrixfilms_v2
{
  "mappings": {
    "_source": {
      "excludes": ["embedding"]
    },
    "properties": {
      "content": { "type": "text", "index": false },
      "embedding": { "type": "dense_vector", "dims": 1536, "index": true },
      "metadata": {
        "properties": {
          "_node_content": { "type": "text", "index": false },
          "_node_type": { "type": "keyword" },
          "doc_id": { "type": "keyword" },
          "document_id": { "type": "keyword" },
          "id": { "type": "long" },
          "ref_doc_id": { "type": "keyword" },
          "title": { "type": "text" }
        }
      }
    }
  }
}

POST _reindex?wait_for_completion
{
  "source": {
    "index": "matrixfilms"
  },
  "dest": {
    "index": "matrixfilms_v2"
  }
}

POST matrixfilms_v2/_flush
```

Now we wait for 30 seconds. I have explained the reason for this wait in my [previous blog post](2024-05-27-optimizing-elasticsearch-disk-usage). Briefly, when during the indexing process the source is modified in any way, Elasticsearch temporarily creates a special field called `_recovery_source`. This field is kept until Elasticsearch ensures that it no longer needs it to perform per-record replication and after that, it is discarded during the merge operation. In our setup it takes 30 seconds. So after 30 seconds, we will run the following command which will merge all segments created during re-indexing while removing this temporary source:

```plaintext
POST matrixfilms_v2/_forcemerge?max_num_segments=1
```

Now we can check the index size:

```plaintext
GET matrixfilms_v2/_stats/store?human&filter_path=indices.*.primaries.store.size
```

```json
{
  "indices": {
    "matrixfilms_v2": {
      "primaries": {
        "store": {
          "size": "15.7mb"
        }
      }
    }
  }
}
```

As you can see, with this simple change, we reduced the index size from 71.8 MB to 15.7 MB, which is a 4.5x reduction in size! Let’s now check that everything still works fine. Back in Python, we can instantiate a new query engine on the new index and execute a query:

```python
# Create LlamaIndex vector store with a new index
vector_store = ElasticsearchStore(
    index_name=ES_INDEX_NAME+"_v2",
    es_url=ES_URL,
    es_user=ES_USER,
    es_password=ES_PASSWORD,
    es_cloud_id=ES_CLOUD_ID,
    es_api_key=ES_API_KEY,
)

# Create index from the store
index = VectorStoreIndex.from_vector_store(vector_store=vector_store)

# Create the query engine
query_engine = index.as_query_engine()

# Query away
print(textwrap.fill(query_engine.query("Who is Neo?").response, width=80))
```

```plaintext
Neo is a main protagonist in The Matrix franchise who was born as Thomas A.
Anderson. He was a former bluepill who was rescued by Morpheus and the crew of
the Nebuchadnezzar, becoming a redpill. Neo was prophesied by The Oracle to be
The One, tasked with freeing humanity from the Matrix and ending the Machine
War. Throughout the series, Neo displays exceptional combat abilities, a direct
connection to the Source, and the power to affect everything connected to it.
His true nature and powers gradually return to him over time, showcasing his
unique abilities and significance in the story.
```

## How Does This Apply to Production?

Obviously, in production, we are not going to create one index and then reindex it into another. Instead, we can either pre-create the index with the desired mapping or use index templates to specify the mapping while allowing LlamaIndex to create the index when needed.

## Conclusion

In this blog post, we showed several simple modifications that allow us to get back almost 80% of our disk space. In the next blog post, we will look at how to shrink it even more by changing the dense vector parameter and see how it affects the quality of our search results. Meanwhile, feel free to experiment with your own datasets and explore further optimizations to suit your specific needs. Happy searching!
For more information or assistance with Elasticsearch and LlamaIndex, don’t hesitate to reach out to Aka`ula Studio LLC.
