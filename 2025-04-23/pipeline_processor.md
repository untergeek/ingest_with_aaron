[Previous: Script Processor](./script_processor.md) | [Next: Error Handling](./error_handling.md)

# Nested Pipelines with the `pipeline` Processor

The [`pipeline` processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/pipeline-processor.html) allows you to call other pipelines, enabling modular and reusable workflows. This approach simplifies complex data processing by breaking it into smaller, reusable components that can be chained together.

## Modular Pipeline Design

Modular pipeline design involves creating small, focused pipelines that perform specific tasks (e.g., parsing logs, enriching data, calculating metrics). These pipelines can be reused across different workflows, reducing duplication and improving maintainability.

## Reusing Common Processor Flows as Stand-Alone Pipelines

Let’s create a `log_parsing` pipeline that parses Apache log lines using a [`grok` processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/grok-processor.html) and reuses the `ingest_lag` pipeline (defined earlier) to calculate the ingestion lag.

```markdown
PUT _ingest/pipeline/log_parsing
{
  "description": "Parses logs and calculates ingest lag",
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": ["%{IP:client.ip} - - \\[%{HTTPDATE:timestamp}\\]"],
        "on_failure": [
          {
            "set": {
              "field": "pipeline_errors.log_parsing.grok",
              "value": "Grok failed: {{ _ingest.on_failure_message }}"
            }
          }
        ]
      }
    },
    {
      "pipeline": {
        "name": "ingest_lag"
      }
    }
  ]
}
```

### Step 1: Simulate

```markdown
POST _ingest/pipeline/log_parsing/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "192.168.1.10 - - [23/Apr/2025:03:20:29 +0000]",
        "@timestamp": "2025-04-23T03:20:29.000Z"
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
          "message": "192.168.1.10 - - [23/Apr/2025:03:20:29 +0000]",
          "@timestamp": "2025-04-23T03:20:29.000Z",
          "client": {
            "ip": "192.168.1.10"
          },
          "event": {
            "ingested": "2025-04-23T03:21:45.779697029Z"
          },
          "lag_in_seconds": 76.781
        },
        "_ingest": {
          "timestamp": "2025-04-23T03:21:45.779697029Z"
        }
      }
    }
  ]
}
```

This pipeline:

- Uses the [`grok` processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/grok-processor.html) to parse the [`message` field](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/grok-processor.html#grok-processor-field) with specified [`patterns`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/grok-processor.html#grok-processor-patterns), extracting the client IP into `client.ip`. If parsing fails, it logs the error in `pipeline_errors.log_parsing.grok` using the [`Set` Processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/set-processor.html).
- Calls the `ingest_lag` pipeline by its [`name`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/pipeline-processor.html#pipeline-processor-name) to add `event.ingested` and calculate `lag_in_seconds` (rounded to three decimal places, e.g., 76.781 seconds for a ~76,781 ms lag).

The [`PUT _ingest/pipeline` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/put-pipeline-api.html) is used to create the pipeline, and the [`Simulate Pipeline` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/simulate-pipeline-api.html) tests it.

## Chaining Pipelines for Complex Workflows

To demonstrate a complex workflow, let’s create a `master_pipeline` that chains the `enrich_user_roles` and `log_parsing` pipelines, enriching documents with user roles and parsing logs with lag calculation.

```markdown
PUT _ingest/pipeline/master_pipeline
{
  "description": "Master pipeline for enrichment, log parsing, and lag calculation",
  "processors": [
    {
      "pipeline": {
        "name": "enrich_user_roles"
      }
    },
    {
      "pipeline": {
        "name": "log_parsing"
      }
    }
  ],
  "on_failure": [
    {
      "set": {
        "field": "pipeline_errors.master_pipeline",
        "value": "Master pipeline failed: {{ _ingest.on_failure_message }}"
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

### Step 2: Simulate

```markdown
POST _ingest/pipeline/master_pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "username": "alice",
        "message": "192.168.1.10 - - [23/Apr/2025:03:20:29 +0000]",
        "@timestamp": "2025-04-23T03:20:29.000Z"
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
          "username": "alice",
          "message": "192.168.1.10 - - [23/Apr/2025:03:20:29 +0000]",
          "@timestamp": "2025-04-23T03:20:29.000Z",
          "user": {
            "role": "admin"
          },
          "client": {
            "ip": "192.168.1.10"
          },
          "event": {
            "ingested": "2025-04-23T03:21:45.779697029Z"
          },
          "lag_in_seconds": 76.781
        },
        "_ingest": {
          "timestamp": "2025-04-23T03:21:45.779697029Z"
        }
      }
    }
  ]
}
```

This pipeline:

- Calls `enrich_user_roles` to add `user.role` (e.g., `admin` for `alice`).
- Calls `log_parsing` to parse the log and calculate the ingestion lag.
- If any processor fails, the `on_failure` block logs the error in `pipeline_errors.master_pipeline` and redirects the document to the `failed_events` index.

If the `username` is not found in the enrich index (e.g., `charlie`), the `enrich_user_roles` pipeline will set `user.role` to `unknown` and log an error in `pipeline_errors.enrich_user_roles.script`, but the pipeline will continue processing.

---

[Previous: Script Processor](./script_processor.md) | [Next: Error Handling](./error_handling.md)