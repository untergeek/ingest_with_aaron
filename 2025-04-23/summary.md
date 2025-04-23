[Previous: Error Handling](./error_handling.md)

# Summary: Key Concepts in Ingest Pipelines 301

Ingest Pipelines 301 has equipped you with advanced techniques to build sophisticated data workflows in Elasticsearch. Let’s recap the key concepts covered in this course:

## Enriching Data with the `enrich` Processor

- **Concept**: The [`enrich` processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/enrich-processor.html) adds external context to documents by matching fields against a dedicated enrich index, enabling seamless data augmentation (e.g., adding user roles to logs).
- **Key Takeaways**:
  - Create and execute enrich policies using the [`Put Enrich Policy` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/put-enrich-policy-api.html) and [`Execute Enrich Policy` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/execute-enrich-policy-api.html) to prepare lookup data for fast processing, specifying [`match_field`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/enrich-processor.html#enrich-processor-match-field) and [`enrich_fields`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/enrich-processor.html#enrich-processor-enrich-fields).
  - Use the [`enrich` processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/enrich-processor.html) to append fields like `user.role` to a [`target_field`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/enrich-processor.html#enrich-processor-target-field).
  - Validate enrichment with a [`script` processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/script-processor.html) to handle missing data, logging errors in `pipeline_errors`.

## Custom Transformations with the `script` Processor

- **Concept**: The [`script` processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/script-processor.html) uses [Painless scripting](https://www.elastic.co/guide/en/elasticsearch/painless/8.18/painless-lang-spec.html) for flexible transformations, such as calculating metrics or manipulating fields.
- **Key Takeaways**:
  - Calculate ingestion lag using `ZonedDateTime` and `ChronoUnit.MILLIS` in the [Painless ingest context](https://www.elastic.co/guide/en/elasticsearch/painless/8.18/painless-ingest-processor-context.html), storing timestamps in ECS-compliant `event.ingested` with a [`source`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/script-processor.html#script-processor-source) script.
  - Handle errors with `on_failure` blocks, logging issues in `pipeline_errors.ingest_lag.script` using the [`Set` Processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/set-processor.html).
  - Round results (e.g., `lag_in_seconds` to three decimal places) for precision.

## Modular Workflows with the `pipeline` Processor

- **Concept**: The [`pipeline` processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/pipeline-processor.html) enables modular workflows by chaining reusable pipelines, simplifying complex data processing.
- **Key Takeaways**:
  - Build stand-alone pipelines like `log_parsing` for specific tasks (e.g., parsing logs with the [`grok` processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/grok-processor.html) using [`patterns`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/grok-processor.html#grok-processor-patterns)).
  - Chain pipelines in a `master_pipeline` by [`name`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/pipeline-processor.html#pipeline-processor-name) to combine enrichment, parsing, and metrics.
  - Use `on_failure` blocks to redirect failed documents to `failed_events` with the [`Set` Processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/set-processor.html).

## Robust Error Handling with Nested Structures and Reindexing

- **Concept**: Advanced error handling uses nested `pipeline_errors` structures, `tags`, and the [`Reindex` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/docs-reindex.html) to track and recover from failures.
- **Key Takeaways**:
  - Log errors in `pipeline_errors.PIPELINE_NAME.PROCESSOR_NAME` for traceability using the [`Set` Processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/set-processor.html) with [`field`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/set-processor.html#set-processor-field) and [`value`](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/set-processor.html#set-processor-value).
  - Use `tags` (e.g., `grok_failed`) and `on_failure` blocks with the [`Append` Processor](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/append-processor.html) to mark and redirect failed documents.
  - Fix pipeline issues (e.g., add [`grok` patterns](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/grok-processor.html#grok-processor-patterns)) and reindex failed documents from `failed_events` to `logs` using the [`Reindex` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/docs-reindex.html).
  - Clean up with the [`Delete By Query` API](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/docs-delete-by-query.html) to maintain a tidy `failed_events` index.

## Next Steps

Apply these techniques to your own Elasticsearch workflows:

- Experiment with enrich policies for custom data augmentation.
- Use [Painless scripts](https://www.elastic.co/guide/en/elasticsearch/painless/8.18/painless-lang-spec.html) for complex transformations.
- Design modular pipelines to streamline processing.
- Implement robust error handling to ensure data integrity.

Thank you for joining Ingest Pipelines 301! Explore more Elasticsearch features in future sessions.

---

[Previous: Error Handling](./error_handling.md)