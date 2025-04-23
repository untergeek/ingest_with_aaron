[Previous: Home](./README.md) | [Next: Script Processor](./script_processor.md)

# The `enrich` Processor: Adding Context to Your Data

The `enrich` processor enhances incoming documents by adding data from a dedicated enrich index based on a matching field, such as appending user roles to log events or geolocation data to IP addresses. An **enrich index** is a specialized Elasticsearch index containing the lookup data (e.g., a mapping of usernames to roles). To use this data, you create an **enrich policy**, which defines how to match incoming documents to the enrich index (via a `match_field`) and which fields to append (via `enrich_fields`). The policy acts as a blueprint for the enrichment process but remains inactive until you **execute** it. Executing the policy generates a hidden, optimized index (a snapshot of the enrich index) that Elasticsearch uses for fast lookups, ensuring efficient performance during ingestion. This step is crucial because it prepares the data for real-time processing, avoiding the need to query the full enrich index repeatedly. The **enrich pipeline** then uses the `enrich` processor to apply the executed policy, matching a specified field in the incoming document (e.g., `username`) against the policy’s `match_field` and appending the specified `enrich_fields` (e.g., `role`) to the document, typically under a `target_field`. This process enables seamless data augmentation without external queries, making it ideal for scenarios requiring contextual enrichment.

## Setting Up an Enrich Index and Policy

First, create an index with enrichment data (e.g., user roles) and define an enrich policy to match documents.

### Step 1: Create an Enrich Index

```markdown
PUT user_roles
{
  "mappings": {
    "properties": {
      "username": { "type": "keyword" },
      "role": { "type": "keyword" }
    }
  }
}
POST user_roles/_doc
{
  "username": "alice",
  "role": "admin"
}
POST user_roles/_doc
{
  "username": "bob",
  "role": "user"
}
```

### Step 2: Create an Enrich Policy

```markdown
PUT _enrich/policy/user_role_policy
{
  "match": {
    "indices": "user_roles",
    "match_field": "username",
    "enrich_fields": ["role"]
  }
}
POST _enrich/policy/user_role_policy/_execute
```

- `match_field`: The field in the enrich index to match against.
- `enrich_fields`: Fields to add to incoming documents.
- `_execute`: Activates the policy.

## Enriching Documents with External Data

Create a pipeline to enrich incoming documents with user roles.

```markdown
PUT _ingest/pipeline/enrich_user_roles
{
  "description": "Enriches documents with user roles",
  "processors": [
    {
      "enrich": {
        "policy_name": "user_role_policy",
        "field": "username",
        "target_field": "user"
      }
    }
  ]
}
```

### Step 3: Simulate

```markdown
POST _ingest/pipeline/enrich_user_roles/_simulate
{
  "docs": [
    {
      "_source": {
        "username": "alice"
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
        "_source": {
          "username": "alice",
          "user": {
            "role": "admin"
          }
        }
      }
    }
  ]
}
```

The `user.role` field is added, enriching the document with the role from the `user_roles` index.

## Validating Enriched Fields

If the `username` doesn’t exist in the enrich index, the `enrich` processor does not add the expected fields (e.g., `user.role`) and continues without failing. To handle cases where the absence of enriched fields is problematic, use a `script` processor to check for the presence of `user.role`. If it’s missing, set a default value and log an error in the `pipeline_errors` structure.

```markdown
PUT _ingest/pipeline/enrich_user_roles
{
  "description": "Enriches documents with user roles and validates enrichment",
  "processors": [
    {
      "enrich": {
        "policy_name": "user_role_policy",
        "field": "username",
        "target_field": "user"
      }
    },
    {
      "script": {
        "source": """
          if (ctx.user?.role == null) {
            ctx.user = ctx.user != null ? ctx.user : new HashMap();
            ctx.user.role = 'unknown';
            ctx.pipeline_errors = ctx.pipeline_errors != null ? ctx.pipeline_errors : new HashMap();
            ctx.pipeline_errors.enrich_user_roles = ctx.pipeline_errors.enrich_user_roles != null ? ctx.pipeline_errors.enrich_user_roles : new HashMap();
            ctx.pipeline_errors.enrich_user_roles.script = 'No enrichment data found for username: ' + ctx.username;
          }
        """
      }
    }
  ]
}
```

### Step 4: Simulate with a Missing Username

```markdown
POST _ingest/pipeline/enrich_user_roles/_simulate
{
  "docs": [
    {
      "_source": {
        "username": "charlie"
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
          "username": "charlie",
          "user": {
            "role": "unknown"
          },
          "pipeline_errors": {
            "enrich_user_roles": {
              "script": "No enrichment data found for username: charlie"
            }
          }
        },
        "_ingest": {
          "timestamp": "2025-04-24T12:00:00.000Z"
        }
      }
    }
  ]
}
```

This pipeline:

- Uses the `enrich` processor to attempt enrichment, which silently skips if no match is found for `charlie`.
- Uses a `script` processor to check if `user.role` is missing (`ctx.user?.role == null`).
- If missing, the script:
  - Initializes the `user` object as a new `HashMap` if it doesn’t exist.
  - Sets `user.role` to `unknown`.
  - Initializes the `pipeline_errors` and `pipeline_errors.enrich_user_roles` as new `HashMap` objects if they don’t exist.
  - Logs the issue under `pipeline_errors.enrich_user_roles.script`.

This approach ensures you can proactively handle non-enriched documents without relying on the `enrich` processor to fail.

---

[Previous: Home](./README.md) | [Next: Script Processor](./script_processor.md)