[Previous: Pipeline Processor](./pipeline_processor.md) | [Next: Summary](./summary.md)

# Advanced Error Handling

Robust error handling ensures pipelines don’t silently drop data. We’ll use nested error structures, handle failures with `tags` and `on_failure`, improve a failing pipeline, and reprocess failed documents using the [`Reindex` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/docs-reindex.html).

## Using a Nested Object Structure for Error Logging

To track errors systematically, we store them in a `pipeline_errors.PIPELINE_NAME.PROCESSOR_NAME` structure. This nested object approach provides clear traceability, identifying which pipeline and processor failed.

## Combine Tags, Processor-Level `on_failure`, and Pipeline-Level `on_failure`

Let’s create a `log_error_handling` pipeline that parses Apache log lines using a [`grok` processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/grok-processor.html), which will fail on logs with an unexpected `ERROR` prefix. We’ll use `tags` to mark failures and redirect failed documents to the `failed_events` index.

```markdown
PUT _ingest/pipeline/log_error_handling
{
  "description": "Pipeline with error handling for log parsing",
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": ["^%{IP:client.ip} - - \\[%{HTTPDATE:timestamp}\\]"],
        "on_failure": [
          {
            "set": {
              "field": "pipeline_errors.log_error_handling.grok",
              "value": "Grok failed: {{ _ingest.on_failure_message }}"
            }
          },
          {
            "append": {
              "field": "tags",
              "value": ["grok_failed"]
            }
          },
          {
            "set": {
              "field": "_index",
              "value": "failed_events"
            }
          }
        ]
      }
    },
    {
      "set": {
        "field": "status",
        "value": "parse_error",
        "if": "ctx.tags != null && ctx.tags.contains('grok_failed')"
      }
    }
  ],
  "on_failure": [
    {
      "set": {
        "field": "pipeline_errors.log_error_handling",
        "value": "Pipeline failed: {{ _ingest.on_failure_message }}"
      }
    },
    {
      "set": {
        "field": "_index",
        "value": "failed_events"
      }
    }
  ]
}
```

### Step 1: Simulate

Simulate the pipeline with a log line containing an unexpected `ERROR` prefix, causing the [`grok` processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/grok-processor.html) to fail.

```markdown
POST _ingest/pipeline/log_error_handling/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "ERROR 192.168.1.10 - - [23/Apr/2025:03:20:29 +0000]",
        "tags": ["parse_log"]
      }
    }
  ]
}
```

**Output:**

```json
{
  "docs": [
    {
      "doc": {
        "_index": "failed_events",
        "_id": "_id",
        "_version": "-3",
        "_source": {
          "message": "ERROR 192.168.1.10 - - [23/Apr/2025:03:20:29 +0000]",
          "tags": ["parse_log", "grok_failed"],
          "pipeline_errors": {
            "log_error_handling": {
              "grok": "Grok parsing failed: no patterns matched"
            }
          },
          "status": "parse_error"
        },
        "_ingest": {
          "timestamp": "2025-04-23T03:21:45.779697029Z"
        }
      }
    }
  ]
}
```

This simulation shows the pipeline:

- Attempts to parse the [`message` field](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/grok-processor.html#grok-processor-field) with specified [`patterns`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/grok-processor.html#grok-processor-patterns), expecting a standard Apache log format starting with an IP address, failing due to the `ERROR` prefix.
- The [`grok` processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/grok-processor.html)’s `on_failure` block:
  - Logs the error in `pipeline_errors.log_error_handling.grok` using the [`Set` Processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/set-processor.html) with [`field`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/set-processor.html#set-processor-field) and [`value`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/set-processor.html#set-processor-value).
  - Adds `grok_failed` to `tags` using the [`Append` Processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/append-processor.html) with [`field`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/append-processor.html#append-processor-field) and [`value`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/append-processor.html#append-processor-value).
  - Redirects the document to `failed_events`.
- Sets `status` to `parse_error` based on `grok_failed` using the [`Set` Processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/set-processor.html).

The [`PUT _ingest/pipeline` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/put-pipeline-api.html) creates the pipeline, and the [`Simulate Pipeline` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/simulate-pipeline-api.html) tests it.

### Step 2: Index a Document and Verify `failed_events`

Index a document to the `logs` index through the `log_error_handling` pipeline, causing it to fail and be redirected to `failed_events`. Verify the index’s creation and contents.

#### Check if `failed_events` Exists (Before)

```markdown
HEAD failed_events
```

**Response (if index doesn’t exist):**

```
404 Not Found
```

#### Index a Document

```markdown
POST logs/_doc?pipeline=log_error_handling
{
  "message": "ERROR 192.168.1.10 - - [23/Apr/2025:03:20:29 +0000]",
  "tags": ["parse_log"]
}
```

**Response:**

```json
{
  "_index": "failed_events",
  "_id": "<generated_id>",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "_seq_no": 0,
  "_primary_term": 1
}
```

The document is sent to the `logs` index using the [`Index` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/docs-index_.html) but redirected to `failed_events` by the [`grok` processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/grok-processor.html)’s `on_failure` block.

#### Check if `failed_events` Exists (After)

```markdown
HEAD failed_events
```

**Response (if index exists):**

```
200 OK
```

#### Search `failed_events` to Verify Contents

```markdown
GET failed_events/_search
{
  "query": {
    "match_all": {}
  }
}
```

**Output:**

```json
{
  "took": 1,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 1,
      "relation": "eq"
    },
    "max_score": 1.0,
    "hits": [
      {
        "_index": "failed_events",
        "_id": "<generated_id>",
        "_score": 1.0,
        "_source": {
          "message": "ERROR 192.168.1.10 - - [23/Apr/2025:03:20:29 +0000]",
          "tags": ["parse_log", "grok_failed"],
          "pipeline_errors": {
            "log_error_handling": {
              "grok": "Grok parsing failed: no patterns matched"
            }
          },
          "status": "parse_error"
        }
      }
    ]
  }
}
```

This confirms the document was indexed in `failed_events` with the expected error details, verified using the [`Search` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/search-search.html).

### Step 3: Simulate an Improved Pipeline

To fix the [`grok` processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/grok-processor.html) failure, simulate an improved pipeline without creating it, adding a pattern to handle `ERROR`-prefixed logs.

```markdown
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "description": "Improved pipeline for log parsing",
    "processors": [
      {
        "grok": {
          "field": "message",
          "patterns": [
            "^%{IP:client.ip} - - \\[%{HTTPDATE:timestamp}\\]",
            "^ERROR %{IP:client.ip} - - \\[%{HTTPDATE:timestamp}\\]"
          ],
          "on_failure": [
            {
              "set": {
                "field": "pipeline_errors.log_error_handling.grok",
                "value": "Grok parsing failed: {{ _ingest.on_failure_message }}"
              }
            },
            {
              "append": {
                "field": "tags",
                "value": ["grok_failed"]
              }
            },
            {
              "set": {
                "field": "_index",
                "value": "failed_events"
              }
            }
          ]
        }
      },
      {
        "set": {
          "field": "status",
          "value": "parse_error",
          "if": "ctx.tags != null && ctx.tags.contains('grok_failed')"
        }
      }
    ]
  },
  "docs": [
    {
      "_source": {
        "message": "ERROR 192.168.1.10 - - [23/Apr/2025:03:20:29 +0000]",
        "tags": ["parse_log"]
      }
    }
  ]
}
```

**Output:**

```json
{
  "docs": [
    {
      "doc": {
        "_index": "_index",
        "_id": "_id",
        "_version": "-3",
        "_source": {
          "message": "ERROR 192.168.1.10 - - [23/Apr/2025:03:20:29 +0000]",
          "tags": ["parse_log"],
          "client": {
            "ip": "192.168.1.10"
          },
          "timestamp": "23/Apr/2025:03:20:29 +0000"
        },
        "_ingest": {
          "timestamp": "2025-04-23T03:21:45.779697029Z"
        }
      }
    }
  ]
}
```

This simulation shows the improved pipeline successfully parses the `ERROR`-prefixed log, extracting `client.ip` and `timestamp` without triggering `on_failure`, using the [`Simulate Pipeline` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/simulate-pipeline-api.html).

### Step 4: Create and Reindex with the Improved Pipeline

Create the improved pipeline and reindex the failed documents from `failed_events` to the `logs` index.

#### Create the Improved Pipeline

```markdown
PUT _ingest/pipeline/log_error_handling
{
  "description": "Improved pipeline for log parsing",
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": [
          "^%{IP:client.ip} - - \\[%{HTTPDATE:timestamp}\\]",
          "^ERROR %{IP:client.ip} - - \\[%{HTTPDATE:timestamp}\\]"
        ],
        "on_failure": [
          {
            "set": {
              "field": "pipeline_errors.log_error_handling.grok",
              "value": "Grok parsing failed: {{ _ingest.on_failure_message }}"
            }
          },
          {
            "append": {
              "field": "tags",
              "value": ["grok_failed"]
            }
          },
          {
            "set": {
              "field": "_index",
              "value": "failed_events"
            }
          }
        ]
      }
    },
    {
      "set": {
        "field": "status",
        "value": "parse_error",
        "if": "ctx.tags != null && ctx.tags.contains('grok_failed')"
      }
    }
  ]
}
```

#### Reindex from `failed_events`

```markdown
POST _reindex
{
  "source": {
    "index": "failed_events",
    "query": {
      "term": {
        "tags": "grok_failed"
      }
    }
  },
  "dest": {
    "index": "logs",
    "pipeline": "log_error_handling"
  }
}
```

**Response (example):**

```json
{
  "took": 10,
  "timed_out": false,
  "total": 1,
  "updated": 0,
  "created": 1,
  "deleted": 0,
  "batches": 1,
  "noops": 0,
  "version_conflicts": 0,
  "retries": {
    "bulk": 0,
    "search": 0
  },
  "throttled_millis": 0,
  "requests_per_second": -1.0,
  "throttled_until_millis": 0,
  "failures": []
}
```

#### Verify Reindexed Documents in `logs`

```markdown
GET logs/_search
{
  "query": {
    "match_all": {}
  }
}
```

**Output:**

```json
{
  "took": 1,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 1,
      "relation": "eq"
    },
    "max_score": 1.0,
    "hits": [
      {
        "_index": "logs",
        "_id": "<generated_id>",
        "_score": 1.0,
        "_source": {
          "message": "ERROR 192.168.1.10 - - [23/Apr/2025:03:20:29 +0000]",
          "tags": ["parse_log"],
          "client": {
            "ip": "192.168.1.10"
          },
          "timestamp": "23/Apr/2025:03:20:29 +0000"
        }
      }
    ]
  }
}
```

The reindexed document is now in `logs`, successfully parsed without errors, verified using the [`Search` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/search-search.html).

### Step 5: Delete Reindexed Documents with Delete By Query

Remove the reindexed documents from `failed_events`:

```markdown
POST failed_events/_delete_by_query
{
  "query": {
    "term": {
      "tags": "grok_failed"
    }
  }
}
```

**Response (example):**

```json
{
  "took": 5,
  "timed_out": false,
  "total": 1,
  "deleted": 1,
  "batches": 1,
  "version_conflicts": 0,
  "noops": 0,
  "retries": {
    "bulk": 0,
    "search": 0
  },
  "throttled_millis": 0,
  "requests_per_second": -1.0,
  "throttled_until_millis": 0,
  "failures": []
}
```

**What’s Happening?**

- The original `log_error_handling` pipeline fails to parse `ERROR`-prefixed logs, redirecting them to `failed_events`.
- The improved pipeline adds a pattern to handle `ERROR`, successfully parsing the log.
- The [`Reindex` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/docs-reindex.html) reprocesses `grok_failed` documents from `failed_events` to `logs` using the improved pipeline, correcting the parsing issue.
- The [`Delete By Query` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/docs-delete-by-query.html) removes the reindexed documents from `failed_events`, keeping the index clean.
- **Important**: Run `_delete_by_query` only after confirming the reindex succeeded (check `logs` for reindexed documents). Monitor the reindex task using `GET _tasks?detailed=true&actions=*reindex` or verify document counts in `logs`.

---

[Previous: Pipeline Processor](./pipeline_processor.md) | [Next: Summary](./summary.md)