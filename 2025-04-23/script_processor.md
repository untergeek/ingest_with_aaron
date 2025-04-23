[Previous: Enrich Processor](./enrich_processor.md) | [Next: Pipeline Processor](./pipeline_processor.md)

# The `script` Processor: Extracting Even More Value

The `script` processor uses Painless scripting to perform custom transformations, offering flexibility for complex logic.

## Understanding the Script Processor and Painless

Painless is Elasticsearch’s secure scripting language. It’s used in the `script` processor to manipulate fields, perform calculations, or apply conditional logic.

## Calculate Ingest Lag Using a Script Processor

Let’s calculate the lag between a document’s event time (`@timestamp`) and ingestion time (`_ingest.timestamp`) in seconds, rounded to three decimal places for millisecond precision, storing the ingestion timestamp in the ECS-compliant `event.ingested` field.

```markdown
PUT _ingest/pipeline/ingest_lag
{ 
  "description": "Add an ingest timestamp and calculate ingest lag", 
  "processors": [ 
    { 
      "set": { 
        "field": "_source.event.ingested", 
        "value": "{{_ingest.timestamp}}" 
      } 
    }, 
    { 
      "script": { 
        "lang": "painless", 
        "source": """ 
            if(ctx.containsKey("event") && ctx.event.containsKey("ingested") && ctx.containsKey("@timestamp")) { 
              def lagMs = ChronoUnit.MILLIS.between(ZonedDateTime.parse(ctx['@timestamp']), ZonedDateTime.parse(ctx.event['ingested']));
              ctx['lag_in_seconds'] = Math.round(lagMs / 1000.0 * 1000) / 1000.0;
            } 
        """,
        "on_failure": [
          {
            "set": {
              "field": "lag_in_seconds",
              "value": -1
            }
          },
          {
            "set": {
              "field": "pipeline_errors.ingest_lag.script",
              "value": "Failed to calculate lag: {{ _ingest.on_failure_message }}"
            }
          }
        ]
      } 
    } 
  ] 
}
```

### Step 1: Simulate

```markdown
POST _ingest/pipeline/ingest_lag/_simulate
{
  "docs": [
    {
      "_source": {
        "@timestamp": "2025-04-23T03:20:29.869Z"
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
          "@timestamp": "2025-04-23T03:20:29.869Z",
          "event": {
            "ingested": "2025-04-23T03:21:45.779697029Z"
          },
          "lag_in_seconds": 75.911
        },
        "_ingest": {
          "timestamp": "2025-04-23T03:21:45.779697029Z"
        }
      }
    }
  ]
}
```

The `lag_in_seconds` field shows a 75.911-second lag, calculated as the difference between `_ingest.timestamp` (`2025-04-23T03:21:45.779697029Z`) and `@timestamp` (`2025-04-23T03:20:29.869Z`), rounded to three decimal places. If the script fails (e.g., due to malformed timestamps or missing fields), the `on_failure` block sets `lag_in_seconds` to `-1` and logs the error in `pipeline_errors.ingest_lag.script`.

## Making It a Stand-Alone Pipeline

This `script` processor can be reused in other pipelines, as we’ll see in later sections.

---

[Previous: Enrich Processor](./enrich_processor.md) | [Next: Pipeline Processor](./pipeline_processor.md)