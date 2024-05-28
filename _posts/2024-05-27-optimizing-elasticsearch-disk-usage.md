---
title: "Optimizing Elasticsearch Disk Usage"
author: imotov
pin: false
categories: [how-tos, intermediate]
tags: [elasticsearch, how-to]
---

## Introduction

When managing a large volume of data, there inevitably comes a time when the disk space consumed by Elasticsearch becomes a concern. In response, you may begin tweaking settings and mappings, or even removing certain fields from the source. But you cannot control what you don’t measure, so you measure the current index size, optimize everything you can, reindex your data, and then discover that... your index size has actually increased instead of decreasing! How could this happen?

To better answer this question, let’s take a look at some example data. For this exercise, we will use a dataset that contains all works of Shakespeare. You can download this dataset at <https://download.elastic.co/demos/kibana/gettingstarted/shakespeare_6.0.json>

## Uploading the Dataset to Elasticsearch

First, let’s create an index with a very simple mapping by running the following command in Kibana:

```javascript
PUT /shakespeare
{
  "mappings": {
    "properties": {
      "type": { "type": "keyword" },
      "line_id": { "type": "integer" },
      "play_name": { "type": "keyword" },
      "speech_number": { "type": "integer" },
      "line_number": { "type": "keyword" },
      "speaker": { "type": "keyword" },
      "text_entry": { "type": "text" }
    }
  }
}
```

Since we cannot load data directly into Elasticsearch from Kibana, switch to the command line and run the following command:

```bash
$ curl -u elastic -H 'Content-Type: application/x-ndjson' -XPOST "es_host:9200/shakespeare/_bulk?pretty" --data-binary @shakespeare_6.0.json
```

This will take a moment. Once the command returns, switch back to Kibana to verify that the data was successfully loaded by running:

```javascript
GET shakespeare/_count
```

You should get a response like this:

```json
{
  "count": 111396,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  }
}
```

## Measuring the Original Index Size

To measure the size of the original index on disk, we need to ensure that Elasticsearch is in a consistent state. This involves clearing the transaction log and merging the index into a single segment by executing the following commands in Kibana:

```plaintext
POST shakespeare/_flush
POST shakespeare/_forcemerge?max_num_segments=1
```

Now we can get the size of our index

```plaintext
GET shakespeare/_stats/store?human&filter_path=indices.*.primaries
```

The response should look something like this:

```json
{
  "indices": {
    "shakespeare": {
      "primaries": {
        "store": {
          "size": "10.4mb",
          "size_in_bytes": 10993583,
          "total_data_set_size": "10.4mb",
          "total_data_set_size_in_bytes": 10993583,
          "reserved": "0b",
          "reserved_in_bytes": 0
        }
      }
    }
  }
}
```

Our index is about 10.4Mb. Now we can start working on optimization of the index size. The records in this index have the following format:

```json
{
  "type": "line",
  "line_id": 4,
  "play_name": "Henry IV",
  "speech_number": 1,
  "line_number": "1.1.1",
  "speaker": "KING HENRY IV",
  "text_entry": "So shaken as we are, so wan with care,"
}
```

The field `line_id` is somewhat redundant, as it repeats the record ID. We can try removing it from the source and not indexing it. Let’s create a new index and reindex data from the original index into the new one:

```plaintext
PUT /shakespeare_v2
{
  "mappings": {
    "_source": {
      "excludes": ["line_id"]
    },
    "properties": {
      "type": { "type": "keyword" },
      "play_name": { "type": "keyword" },
      "speech_number": { "type": "integer" },
      "line_number": { "type": "keyword" },
      "speaker": { "type": "keyword" },
      "text_entry": { "type": "text" }
    }
  }
}
POST shakespeare_v2/_flush
POST shakespeare_v2/_forcemerge?max_num_segments=1
```

## Checking the New Index Size

Now, let’s check the size of the new index:

```plaintext
GET shakespeare_v2/_stats/store?human&filter_path=indices.*.primaries
```

... and be surprised by the response:

```json
{
  "indices": {
    "shakespeare_v2": {
      "primaries": {
        "store": {
          "size": "12.3mb",
          "size_in_bytes": 12962595,
          "total_data_set_size": "12.3mb",
          "total_data_set_size_in_bytes": 12962595,
          "reserved": "0b",
          "reserved_in_bytes": 0
        }
      }
    }
  }
}
```

We removed a field from an 10.4 MB index and ended up with a 12.3 MB index. How did that happen?

## Investigating the Disk Space Increase

Let’s investigate why our index size increased despite our optimizations. To figure out where the disk space went, we will use the [Disk Usage API](https://www.elastic.co/guide/en/elasticsearch/reference/8.13/indices-disk-usage.html). Analyzing the disk usage of an index can be resource-intensive on large indices, but it should be quick and insightful for our small index.

Run the following command:

```plaintext
POST shakespeare_v2/_disk_usage?run_expensive_tasks=true&filter_path=*.fields.*.total
```

The response should look something like this:

```json
{
  "shakespeare_v2": {
    "fields": {
      "_id": { "total": "916.4kb" },
      "_primary_term": { "total": "0b" },
      "_recovery_source": { "total": "3.9mb" },
      "_seq_no": { "total": "341.6kb" },
      "_source": { "total": "3.6mb" },
      "_version": { "total": "0b" },
      "line_id": { "total": "342.1kb" },
      "line_number": { "total": "524.6kb" },
      "play_name": { "total": "84.7kb" },
      "speaker": { "total": "277.9kb" },
      "speech_number": { "total": "407kb" },
      "text_entry": { "total": "1.9mb" },
      "type": { "total": "42.9kb" }
    }
  }
}
```

## Comparing Disk Usage

If we compare this with the output of the same command for the original index, we find the difference:

```json
{
  "shakespeare": {
    "fields": {
      "_source": { "total": "5.5mb" },
  },
  "shakespeare_v2": {
    "fields": {
      "_recovery_source": { "total": "3.9mb" },
      "_source": { "total": "3.6mb" },
    }
  }
}
```

In the original index, `_source` took up 5.5 MB. In the new index we have two sources: `_recovery_source` and `_source`, which together take 7.5 MB. This accounts for the increase in index size. We found the culprit, but what is `_recovery_source` and how do we get rid of it?

## Understanding \_recovery_source

The `_recovery_source` is an undocumented temporary field automatically added when the original source is modified or disabled to facilitate [history retention](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-history-retention.html). To efficiently recover shards in certain situations, such as shards going offline and for cross-cluster replication, Elasticsearch replays indexing operations for individual documents. To do this, Elasticsearch needs to have access to the original document which means it has to:

1.  Prevent deleted records from fully disappearing until the shard is recovered or replication is finished.
2.  Have access to the **original** source.

Since we modified the source stored in the `_source` by removing a field from it, Elasticsearch no longer has the original document stored in the index, so it adds a special hidden `_recovery_source` containing the entire source of the original document on the top of the `_source` that contains everything except the `line_id` field. Thanks to compression, the overall size grows only slightly instead of doubling.

This field is temporary and, by default, is designed to exist for only 12 hours to allow sufficient time for shard recovery and replication. After this period, the field is typically discarded during segment merge operations. However, there is a catch: the mere presence of `_recovery_source` fields does not trigger a merge. If no other conditions initiate a merge, these fields may remain in the index indefinitely.

In real-life indices that persist for many days, this is not a significant issue, as these indices typically undergo multiple merges, leaving only very small, fresh segments with the `_recovery_source` field. However, this poses a considerable challenge for our benchmark because it prevents us from accurately estimating the disk size savings.

## Removing `_recover_source` for benchmarking

To estimate the disk size savings, we need to address the `_recovery_source` issue. One option is to wait until the next day when all leases have expired. However, a more efficient workaround for benchmarking purposes is to decrease the history retention interval. Let’s create another index, but this time set the `index.soft_deletes.retention_lease.period` setting to a very short duration:

```plaintext
PUT /shakespeare_v3
{
  "mappings": {
    "_source": {
      "excludes": ["line_id"]
    },
    "properties": {
      "type": { "type": "keyword" },
      "play_name": { "type": "keyword" },
      "speech_number": { "type": "integer" },
      "line_number": { "type": "keyword" },
      "speaker": { "type": "keyword" },
      "text_entry": { "type": "text" }
    }
  },
  "settings": {
    "index.soft_deletes.retention_lease.period": "100ms"
  }
}

POST _reindex?wait_for_completion=true
{
  "source": {
    "index": "shakespeare"
  },
  "dest": {
    "index": "shakespeare_v3"
  }
}

POST shakespeare_v3/_flush
```

Now, we wait about a minute. Even though we set the retention lease to 100ms, it can take longer. If we run the force merge too soon, the `_recovery_source` might still make it into the merged segment. After a minute, run:

```plaintext
POST shakespeare_v3/_forcemerge?max_num_segments=1
```

Finally, we check the index size:

```plaintext
GET shakespeare_v3/_stats/store?human&filter_path=indices.*.primaries
```

We should see the expected reduction in size to 9.8 MB:

```json
{
  "indices": {
    "shakespeare_v3": {
      "primaries": {
        "store": {
          "size": "9.8mb",
          "size_in_bytes": 10343501,
          "total_data_set_size": "9.8mb",
          "total_data_set_size_in_bytes": 10343501,
          "reserved": "0b",
          "reserved_in_bytes": 0
        }
      }
    }
  }
}
```

## Conclusion

Optimizing index size in Elasticsearch is not always straightforward. As we’ve seen, source can take a significant disk space and removing fields from it can lead to unexpected increases in disk usage due to features like \_recovery_source. Understanding these internal mechanisms is crucial for effective optimization.

Feel free to experiment with your own datasets and explore further optimizations to suit your specific needs. Happy searching! For more information or assistance with Elasticsearch, don’t hesitate to [reach out to us](/contact) at Aka`ula Studio LLC.
