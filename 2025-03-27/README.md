# Ingest Series: Ingest Pipelines 201

Ingest with Aaron: 2025-03-27

---

# Ingest Pipelines 201: Advanced Data Transformation with Elasticsearch

## Session Overview

In this session, you’ll learn advanced Elasticsearch Ingest Pipeline techniques to transform, secure, and manage complex data workflows. Using Kibana Dev Tools, you’ll learn to parse JSON, anonymize or redact sensitive data, use tags to conditionally execute processors to route data, and handle errors effectively—all while ensuring your pipelines are robust and compliant.

---

# Create and modify Ingest Pipelines using the Kibana Dev Tools (or Developer tools) Console

### Step 1: Navigate to the Dev Tools console

Start by navigating to **Dev Tools** from the sidebar menu. This is where you’ll write and test (well, `_simulate`) your pipelines using the REST API.

In the lower-left corner of your Kibana you may be able to navigate directly to **Developer tools** if your UI looks like this.

![New UI](image/image.png)


Otherwise you may need to expand **Management** to see **Dev Tools**

![Older UI](image/image-1.png)

Navigate to **Dev Tools** or **Developer tools** for the next steps.

---

### Step 2: Create a basic pipeline

Walk them through creating a simple pipeline with a hands-on example. Use the `PUT _ingest/pipeline/<pipeline-name>` API in the Dev Tools Console. Here’s an example:

```
PUT _ingest/pipeline/sample-pipeline
{
  "description": "A pipeline to uppercase a field and set a default value",
  "processors": [
    {
      "uppercase": {
        "field": "username"
      }
    },
    {
      "set": {
        "field": "status",
        "value": "processed"
      }
    }
  ]
}
```

**What am I seeing?**

- `PUT _ingest/pipeline/sample-pipeline`: Creates a pipeline named "sample-pipeline".
- `description`: A human-readable note about what the pipeline does.
- `processors`: The list of actions:
    - `uppercase` converts the "username" field to uppercase
    - `set` adds a "status" field with the value "processed".

**Run it**: Paste this into **Dev Tools** and execute it.

**Verify**: Check your work by running `GET _ingest/pipeline/sample-pipeline` and see the pipeline definition returned.

---

### Step 3: Test the pipeline with sample data

You can test your pipeline using the `_simulate` API. This lets you see the pipeline in action without needing to index real data. You saw the `_simulate` API in the Ingest Pipelines 101 session, you just didn't know that's what it was.

```
POST _ingest/pipeline/sample-pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "username": "john_doe"
      }
    }
  ]
}
```

After running this, you'll see the transformed document:

```
{
  "docs": [
    {
      "doc": {
        "_source": {
          "username": "JOHN_DOE",
          "status": "processed"
        }
      }
    }
  ]
}
```

Note that "username" is now uppercase and "status" was added.

---

### Step 4: Simulate inline

You can also simulate a pipeline without having to create a pipeline. It's a bit different format here:

```
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
      {
        "uppercase": {
          "field": "username"
        }
      },
      {
        "set": {
          "field": "status",
          "value": "processed"
        }
      }
    ]
  },
  "docs": [
    {
      "_source": {
        "username": "john_doe"
      }
    }
  ]
}
```

In `POST _ingest/pipeline/_simulate` we omit the pipeline name, and in the body, we encapsulate all of `processors` within `pipeline: {}`. Then we put `docs` inline, but after the `processors: {}` portion.

If we run this, we note that the results are the same, but we didn't create a pipeline.

---

### Step 5: Modify an existing pipeline

Modifying a pipeline is as simple as re-running a `PUT` command with updated configuration. For example, let’s add a `remove` processor to delete a field:

```
PUT _ingest/pipeline/sample-pipeline
{
  "description": "Updated pipeline with field removal",
  "processors": [
    {
      "uppercase": {
        "field": "username"
      }
    },
    {
      "set": {
        "field": "status",
        "value": "processed"
      }
    },
    {
      "remove": {
        "field": "temporary_field"
      }
    }
  ]
}
```

This completely overwrites the existing pipeline, so you need to include all processors, you intend to keep.

Iterate and test with `_simulate` to validate pipeline changes:

**Simulate:**

```
POST _ingest/pipeline/sample-pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "username": "john_doe",
        "temporary_field": "some value"
      }
    }
  ]
}
```

The output look the same, but will have removed `temporary_field`.

---

# Expand your pipeline knowledge

## The `json` processor

### Step 1: Set up a simple pipeline with the `json` processor

Let's create a pipeline with the `json` processor. This will be a basic example where the JSON string is in a field called `data`:

```
PUT _ingest/pipeline/json-parsing-pipeline
{
  "description": "Parses JSON from the 'data' field",
  "processors": [
    {
      "json": {
        "field": "data",
        "target_field": "parsed_data"
      }
    }
  ]
}
```

**What am I seeing?**

- `field`: The source field containing the JSON string.
- `target_field`: Where the parsed JSON fields will go (_Optional._ Defaults to the root of the document if omitted).

**Run it**: Paste this into **Dev Tools** and execute it.

---

### Step 2: Test with `_simulate`

Now that we are getting the hang of the using the `_simulate` API, let's use it to feed some sample JSON data to our `json-parsing-pipeline`:

```
POST _ingest/pipeline/json-parsing-pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "data": "{\"name\": \"Alice\", \"age\": 30}"
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_source": {
          "data": "{\"name\": \"Alice\", \"age\": 30}",
          "parsed_data": {
            "name": "Alice",
            "age": 30
          }
        }
      }
    }
  ]
}
```

The JSON string in `data` was parsed into a structured object under `parsed_data`. The original `data` field stays unless you remove it later (remember the `remove` processor from the 101 session?)

---

### Step 3: Overwriting the original field

You can parse JSON directly into the document root by omitting `target_field`:`
`

```
PUT _ingest/pipeline/json-parsing-pipeline
{
  "description": "Parses JSON and overwrites the source field",
  "processors": [
    {
      "json": {
        "field": "data"
      }
    }
  ]
}
```

**Simulate:**

```
POST _ingest/pipeline/json-parsing-pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "data": "{\"name\": \"Bob\", \"age\": 25}"
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_source": {
          "name": "Bob",
          "age": 25
        }
      }
    }
  ]
}
```

The `data` field is gone, and its contents are now top-level fields. This is cleaner if you don’t need the original JSON.

---

### Step 4: Handle nested JSON

The `json` processor can also handle complex JSON with nested objects:

```
PUT _ingest/pipeline/json-parsing-pipeline
{
  "description": "Parses nested JSON",
  "processors": [
    {
      "json": {
        "field": "data",
        "target_field": "parsed"
      }
    }
  ]
}
```

**Simulate:**

```
POST _ingest/pipeline/json-parsing-pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "data": "{\"user\": {\"name\": \"Charlie\", \"details\": {\"age\": 35, \"city\": \"New York\"}}}"
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_source": {
          "data": "{\"user\": {\"name\": \"Charlie\", \"details\": {\"age\": 35, \"city\": \"New York\"}}}",
          "parsed": {
            "user": {
              "name": "Charlie",
              "details": {
                "age": 35,
                "city": "New York"
              }
            }
          }
        }
      }
    }
  ]
}
```

As you can see, the nested structure is preserved, and you can access fields like `parsed.user.name` or `parsed.user.details.city`.

---

## Using `on_failure` 

We touched briefly on the `on_failure` directive in the Ingest Pipelines 101 session. The `on_failure` block defines what happens when a processor in the pipeline fails (e.g., due to a missing field, invalid data, or a processing error). It’s like a safety net that prevents the entire pipeline from crashing and lets you handle errors in a controlled way. Think of it like a backup plan—if something goes wrong, `on_failure` steps in to fix it or log it instead of just giving up, which could bounce your documents back to the client that sent them. If that client cannot or will not receive the response, then your documents could end up just disappearing, or getting "dropped on the floor."

When using the Ingest Pipeline UI, you have limited control over what happens within a processor, and `on_failure` was only available at the pipeline level. Using the API in the Dev Tools console allows us much more flexibility in how we can use `on_failure` both within a processor directive and for the whole pipeline.

---

### Step 1: Set up a pipeline with a place to fail

Create a basic pipeline where a failure is likely so you can see how `on_failure` works.  In this example, we'll use the `convert` processor, which changes a field’s data type. This makes the failure easy to guarantee—we simply set up an impossible conversion.`
`

```
PUT _ingest/pipeline/demo-pipeline
{
  "description": "A pipeline with a risky conversion",
  "processors": [
    {
      "convert": {
        "field": "age",
        "type": "integer"
      }
    }
  ]
}
```

This pipeline tries to convert the `age`  field to an integer. If `age`  is a string like "twenty" or missing, it’ll fail.

---

### Step 2: Test the pipeline without `on_failure`

Use the `_simulate` API with bad data:

```
POST _ingest/pipeline/demo-pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "age": "not_a_number"
      }
    }
  ]
}
```

**Output**: (note the error)

```
{
  "docs": [
    {
      "error": {
        "root_cause": [
          {
            "type": "illegal_argument_exception",
            "reason": "unable to convert [not_a_number] to integer"
          }
        ],
        "type": "illegal_argument_exception",
        "reason": "unable to convert [not_a_number] to integer",
        "caused_by": {
          "type": "number_format_exception",
          "reason": "For input string: \"not_a_number\""
        }
      }
    }
  ]
}
```

Without `on_failure`, the pipeline stops, and the document isn’t processed. This is where `on_failure` comes in handy.

---

### Step 3: Add `on_failure` to handle the error

Now, let's modify the pipeline to include an `on_failure` block. In this example, we'll set a default value of `-1` and log the issue in the field `error_message`:

```
PUT _ingest/pipeline/demo-pipeline
{
  "description": "A pipeline with error handling",
  "processors": [
    {
      "convert": {
        "field": "age",
        "type": "integer",
        "on_failure": [
          {
            "set": {
              "field": "age",
              "value": -1
            }
          },
          {
            "set": {
              "field": "error_message",
              "value": "Failed to convert age to integer: {{ _ingest.on_failure_message }}"
            }
          }
        ]
      }
    }
  ]
}
```

**What am I seeing?**

- `on_failure`: A list of processors that run if the `convert` fails.
- The first `set`: Assigns `age`  a default value of `-1`.
- The second `set`: Adds an `error_message` field with the failure reason, using the `{{ _ingest.on_failure_message }}` variable to capture the error details.
    - `{{ }}` is mustache templating. In this case we're taking advantage of the fact that Ingest Pipelines have internal, hidden fields and values as part of the pipeline metadata. They can be accessed using dotted notation, and the metadata field is `_ingest`. So here, we are accessing the failure message generated when the processor failed and including it inline.

**Run it**: Paste this into **Dev Tools** and execute it.

---

### Step 4: Test the updated pipeline

Simulate it again with the same bad data:

```
POST _ingest/pipeline/demo-pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "age": "not_a_number"
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_index": "_index",
        "_version": "-3",
        "_id": "_id",
        "_source": {
          "error_message": "Failed to convert age to integer: For input string: \\\"not_a_number\\\"",
          "age": -1
        },
        "_ingest": {
          "timestamp": "2025-03-27T15:52:11.492567812Z"
        }
      }
    }
  ]
}
```

Note that the pipeline didn’t fail—it processed the document, set `age` to `-1`, and logged the error in the `error_message` field. This document would still be indexed, albeit with those values.

---

### Step 5: Configuring `on_failure` at the pipeline level

In the Pipelines 101 session we learned how `on_failure` can be defined at the top level of the pipeline, catching failures from any processor. Since we are using the API this time, we'll go cover it again.

```
PUT _ingest/pipeline/demo-pipeline
{
  "description": "Pipeline-level error handling",
  "processors": [
    {
      "convert": {
        "field": "age",
        "type": "integer"
      }
    }
  ],
  "on_failure": [
    {
      "set": {
        "field": "pipeline_error",
        "value": "Pipeline failed: {{ _ingest.on_failure_message }}"
      }
    }
  ]
}
```

Simulate using the same `docs` from Step 4. 

**Output:**

```
{
  "docs": [
    {
      "doc": {
        "_index": "_index",
        "_version": "-3",
        "_id": "_id",
        "_source": {
          "age": "not_a_number",
          "pipeline_error": "Pipeline failed: For input string: \\\"not_a_number\\\""
        },
        "_ingest": {
          "timestamp": "2025-03-27T15:52:56.732292253Z"
        }
      }
    }
  ]
}
```

Because this `on_failure` is at the top, or root level of the pipeline, it will catch any failure in any processor. Because of that, we only log the `_ingest.on_failure_message` to the field `pipeline_error`. We only have one processor in this example, but if there were others, and one besides our `convert` processor were to fail, we wouldn't dare to set `age` to `-1` because we wouldn't know exactly which processor failed, or why. 

This is why being able to set `on_failure` at the processor level is powerful.

Watch for subsequent examples to make use of `on_failure`.

---

## The `fingerprint` processor

The `fingerprint` processor can help anonymize data. Why is this an important feature? Obscuring personally identifiable information (PII) is an important way to maintain user privacy. By using the hash of a field value we can obscure sensitive data (e.g., names, emails, or IDs) so they can’t be tied back to individuals, but the data remains usable for analysis. Ingest Pipelines can do this automatically as data is indexed.

In this example we’ll replace an email like `alice@example.com` with a hashed value.

### Step 1: Set up a simple `fingerprint`  pipeline

The `fingerprint` processor creates a consistent hash (e.g., SHA-256) of a field’s value. As mentioned already, this is a great way to anonymize data while preserving uniqueness:

```
PUT _ingest/pipeline/anonymize-email
{
  "description": "Anonymizes email field with a hash",
  "processors": [
    {
      "fingerprint": {
        "fields": ["email"],
        "target_field": "email_hashed",
        "method": "SHA-256"
      }
    }
  ]
}
```

**What am I seeing?**

- `fields`: The source field or fields to anonymize.
- `target_field`: Where the hashed value goes (_Optional_. Default is `fingerprint`).
- `method`: The hashing algorithm (e.g., SHA-256, MD5).

**Run it**: Paste this into **Dev Tools** and execute it.

---

### Step 2: Test with `_simulate`

Test the new pipeline with some sample data:

**Simulate:**

```
POST _ingest/pipeline/anonymize-email/_simulate
{
  "docs": [
    {
      "_source": {
        "email": "alice@example.com"
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_index": "_index",
        "_version": "-3",
        "_id": "_id",
        "_source": {
          "email": "alice@example.com",
          "email_hashed": "JHzr7YtQDgNh2W81fWC3O5mHgIN0al7zkppLx8gzzH8="
        },
        "_ingest": {
          "timestamp": "2025-03-27T15:57:45.805055307Z"
        }
      }
    }
  ]
}
```

The email is now a fixed-length hash. It’s anonymized but still unique to “alice@example.com” (same input = same hash).

---

### Step 3: Overwrite the original field

By removing the `target_field` setting, the `hash` processor will replace the original field value:

```
PUT _ingest/pipeline/anonymize-email
{
  "description": "Anonymizes email by overwriting it",
  "processors": [
    {
      "fingerprint": {
        "fields": ["email"],
        "target_field": "email",
        "method": "SHA-256"
      }
    }
  ]
}
```

Simulate:

```
POST _ingest/pipeline/anonymize-email/_simulate
{
  "docs": [
    {
      "_source": {
        "email": "bob@example.com"
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_index": "_index",
        "_version": "-3",
        "_id": "_id",
        "_source": {
          "email": "8r227/Qy+Qi4DypBWZY3U3jXwQHiAWYKusRfaSaZMfQ="
        },
        "_ingest": {
          "timestamp": "2025-03-27T15:59:23.896634811Z"
        }
      }
    }
  ]
}
```

The original email is gone, replaced by the hash.

---

## The `redact` processor

Where the `fingerprint` processor creates a hash of one or more fields, the `redact` processor uses built-in or custom `grok` patterns to match a substring pattern and replace it with a placeholder.

Example: Using the `redact` processor, `My SSN is 123-45-6789` will become `My SSN is <ssn>`.

---

### Step 1: Set up a basic redaction pipeline

Introduce the `redact` processor with a built-in pattern (e.g., for US Social Security Numbers):

```
PUT _ingest/pipeline/redact-ssn
{
  "description": "Redacts SSNs from a text field",
  "processors": [
    {
      "redact": {
        "field": "message",
        "pattern_definitions": {
          "SSN": "(\\d{3})(?:-(\\d{2})-|\\.(\\d{2})\\.|\\s(\\d{2})\\s)(\\d{4})"
        },
        "patterns": [
          "%{SSN:ssn}"
        ]
      }
    }
  ]
}
```

**What am I seeing?**

- `field`: The source field containing text to redact.
- `pattern_definitions`: A set of key value pairs defining a named `grok` pattern
    - `SSN` is the name of the pattern
    - `(\\d{3})(?:-(\\d{2})-|\\.(\\d{2})\\.|\\s(\\d{2})\\s)(\\d{4})`  Matches SSN patterns of 3, 2, then 4 digits separated by either dashes, dots, or spaces.
- `patterns`: A list of patterns to match (`SSN` is our named `grok` pattern). Default behavior: Matched text is replaced with what follows the colon (`:`), in this case`<ssn>`.

**Run it**: Paste this into **Dev Tools** and execute it.

---

### Step 2: Test with `_simulate`

```
POST _ingest/pipeline/redact-ssn/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "My SSN is 123-45-6789, please keep it safe."
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_index": "_index",
        "_version": "-3",
        "_id": "_id",
        "_source": {
          "message": "My SSN is <ssn>, please keep it safe."
        },
        "_ingest": {
          "timestamp": "2025-03-27T15:59:59.035249852Z"
        }
      }
    }
  ]
}
```

As you can see, the SSN was replaced with `<ssn>`, but the rest of the text stays intact.

---

### Step 3: Create custom patterns

Redact custom sensitive data (e.g., credit card numbers) by adding your own named regex pattern:

```
PUT _ingest/pipeline/redact-credit-card
{
  "description": "Redacts credit card numbers",
  "processors": [
    {
      "redact": {
        "field": "message",
        "pattern_definitions": {
          "CARD": "(?:\\d[ -]*?){13,16}"
        },
        "patterns": [
          "%{CARD:card}"
        ]
      }
    }
  ]
}
```

**What am I seeing?**

- `pattern_definitions`: A set of key value pairs defining a named `grok` pattern
    - `CARD` is the name of the pattern
    - `(?:\\d[ -]*?){13,16}`: Matches 13-16 digits, optionally separated by spaces or hyphens (a basic credit card pattern).
- `%{CARD:card}` `CARD` matches the named `grok` pattern and `card` becomes the field name. In the case of the `redact` processor, it's the value replacing the thing being redacted.

**Simulate:**

```
POST _ingest/pipeline/redact-credit-card/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "My card is 1234-5678-9012-3456."
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_index": "_index",
        "_version": "-3",
        "_id": "_id",
        "_source": {
          "message": "My card is <card>."
        },
        "_ingest": {
          "timestamp": "2025-03-27T16:01:10.069722677Z"
        }
      }
    }
  ]
}
```

---

### Step 4: Redact multiple patterns in one processor

The `redact` processor is not limited to a single pattern. It can handle multiple types of sensitive data in one field (you will need one `redact` processors per field).

```
PUT _ingest/pipeline/redact-multi
{
  "description": "Redacts SSNs and credit cards",
  "processors": [
    {
      "redact": {
        "field": "message",
        "patterns": [
          "%{SSN:ssn}",
          "%{CARD:creditcard}"
        ],
        "pattern_definitions": {
          "SSN": "(\\d{3})(?:-(\\d{2})-|\\.(\\d{2})\\.|\\s(\\d{2})\\s)(\\d{4})",
          "CARD": "(?:\\d[ -]*?){13,16}"
        }
      }
    }
  ]
}
```

**Simulate:**

```
POST _ingest/pipeline/redact-multi/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "SSN: 123-45-6789, Card: 1234-5678-9012-3456"
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_index": "_index",
        "_version": "-3",
        "_id": "_id",
        "_source": {
          "message": "SSN: <ssn>, Card: <creditcard>"
        },
        "_ingest": {
          "timestamp": "2025-03-27T16:01:44.739654889Z"
        }
      }
    }
  ]
}
```

---

## The `gsub` processor

The `gsub` processor searches for a regular expression pattern in a field and replaces all matches with a specified string. Unlike the `redact` processor, which uses predefined placeholders, `gsub` gives you full control over the replacement text.

For example, we can turn `Phone: 123-456-7890` into `Phone: XXX-XXX-XXXX` with a custom regex.

---

### Step 1: Set up a basic `gsub` pipeline

Show how to create a pipeline that redacts a phone number:

```
PUT _ingest/pipeline/redact-phone
{
  "description": "Redacts phone numbers with gsub",
  "processors": [
    {
      "gsub": {
        "field": "message",
        "pattern": "\\d{3}-\\d{3}-\\d{4}",
        "replacement": "XXX-XXX-XXXX"
      }
    }
  ]
}
```

**What am I seeing?**

- `field`: The field to search (e.g., `message`).
- `pattern`: A regex for US phone numbers (`XXX-XXX-XXXX`).
- `replacement`: The string to replace each match with.

**Run it**: Paste this into **Dev Tools** and execute it.

---

### Step 2: Test with `_simulate`

**Simulate:**

```
POST _ingest/pipeline/redact-phone/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "Call me at 123-456-7890."
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_source": {
          "message": "Call me at XXX-XXX-XXXX."
        }
      }
    }
  ]
}
```

As you can see, the phone number was replaced with `XXX-XXX-XXXX`, redacting it while keeping the message readable.

---

### Step 3: Redact partial patterns

We can also redact a part of a pattern. For example, we might only want to mask the first 6 digits of a phone number:

```
PUT _ingest/pipeline/redact-partial-phone
{
  "description": "Redacts first 6 digits of phone numbers",
  "processors": [
    {
      "gsub": {
        "field": "message",
        "pattern": "(\\d{3}-\\d{3})-(\\d{4})",
        "replacement": "XXX-XXX-$2"
      }
    }
  ]
}
```

**What am I seeing?**

- `(\\d{3}-\\d{3})-(\\d{4})`: Captures the first 6 digits in group 1 and last 4 in group 2.
- `XXX-XXX-$2`: Replaces the match, keeping the second group (last 4 digits).

Going into capture groups might be a bit beyond the scope of this session. Suffice to say that in regular expressions, parenthesis identify capture groups. The first group in this case is `(\\d{3}-\\d{3})` which is the first 3 digits, the dash, and the next 3 digits. The second capture group is the last 4 digits.

**Simulate:**

```
POST _ingest/pipeline/redact-partial-phone/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "Number: 123-456-7890"
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_source": {
          "message": "Number: XXX-XXX-7890"
        }
      }
    }
  ]
}
```

Using `gsub` allows us to keep some data (last 4 digits) for context while redacting the rest.

---

### Step 4: Handle multiple patterns

We can also chain `gsub` processors to redact different patterns (e.g., phone numbers and emails):

```
PUT _ingest/pipeline/redact-multi
{
  "description": "Redacts phones and emails",
  "processors": [
    {
      "gsub": {
        "field": "message",
        "pattern": "\\d{3}-\\d{3}-\\d{4}",
        "replacement": "[PHONE]"
      }
    },
    {
      "gsub": {
        "field": "message",
        "pattern": "[\\w\\.]+@[\\w\\.]+",
        "replacement": "[EMAIL]"
      }
    }
  ]
}
```

**What am I seeing?**

- First `gsub`: Redacts phone numbers.
- Second `gsub`: Redacts basic email addresses (`something@domain.com`).

**Simulate:**

```
POST _ingest/pipeline/redact-multi/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "Contact: 123-456-7890 or alice@example.com"
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_source": {
          "message": "Contact: [PHONE] or [EMAIL]"
        }
      }
    }
  ]
}
```

---

## Use `tags` to help route data

The `tags` field is an array of strings (e.g., `["log", "sensitive", "urgent"]`) that acts like metadata. You can use it to control which processors apply to a document based on its tags, making pipelines reusable and adaptable.

Let's build an example pipeline: If a document has the tag `sensitive`, we’ll redact it; if it has `uppercase`, we’ll capitalize a field.

---

### Step 1: Set up a pipeline with conditional logic

Create a pipeline that checks the `tags` array and applies processors conditionally:

```
PUT _ingest/pipeline/tags-based-processing
{
  "description": "Processes data based on tags",
  "processors": [
    {
      "uppercase": {
        "field": "message",
        "if": "ctx.tags != null && ctx.tags.contains('uppercase')"
      }
    },
    {
      "redact": {
        "field": "message",
        "patterns": ["%{SSN:ssn}"],
        "pattern_definitions": {
          "SSN": "(\\d{3})(?:-(\\d{2})-|\\.(\\d{2})\\.|\\s(\\d{2})\\s)(\\d{4})"
        },
        "if": "ctx.tags != null && ctx.tags.contains('sensitive')"
      }
    }
  ]
}
```

**What am I seeing?**

- `if`: A Painless script condition.
- `ctx.tags != null`: Ensures `tags` exists to avoid null errors
- `ctx.tags.contains('uppercase')`: Checks if the array includes the string "uppercase".
- Processors: 
    - `uppercase` runs if "uppercase" is in `tags`
    - `redact` runs if "sensitive" is in `tags`.

**Run it**: Paste this into **Dev Tools** and execute it.

---

### Step 2: Test with `_simulate`

Let's see how it works with different values in `tags`:

**Test 1: "uppercase" tag**

```
POST _ingest/pipeline/tags-based-processing/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "hello world",
        "tags": ["log", "uppercase"]
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_source": {
          "message": "HELLO WORLD",
          "tags": ["log", "uppercase"]
        }
      }
    }
  ]
}
```

Only the `uppercase` processor ran because "uppercase" was in `tags`.

**Test 2: "sensitive" tag**

```
POST _ingest/pipeline/tags-based-processing/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "my ssn is 123-45-6789",
        "tags": ["sensitive"]
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_source": {
          "message": "my ssn is <ssn>",
          "tags": ["sensitive"]
        }
      }
    }
  ]
}
```

As before, only when the `if` condition is matched will the processor execute. This time the `redact` processor acted because "sensitive" is in `tags`.

**Test 3: No matching tags**

```
POST _ingest/pipeline/tags-based-processing/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "just text",
        "tags": ["log"]
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_source": {
          "message": "just text",
          "tags": ["log"]
        }
      }
    }
  ]
}
```

No processors ran this time because neither condition matched.

---

### Step 3: Combine multiple tags - part 1

With multiple tags we can trigger different processors:

```
POST _ingest/pipeline/tags-based-processing/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "secret info ssn is 123-45-6789",
        "tags": ["uppercase", "sensitive"]
      }
    }
  ]
}
```

**Output**:

```
{
  "docs": [
    {
      "doc": {
        "_source": {
          "message": "SECRET INFO SSN IS <ssn>",
          "tags": ["uppercase", "sensitive"]
        }
      }
    }
  ]
}
```

Both conditions were true. The `uppercase` processor ran first, and then `redact`, which is why the `<ssn>` is lowercase still. Order matters. Let's see what happens in a slightly different example.

---

### Step 4: Combine multiple tags - part 2

As a demonstration, we're going to change the previous pipeline a bit to show what _could_ happen with sequential processors and matching `tags`:

```
PUT _ingest/pipeline/tags-based-processing
{
  "description": "Processes data based on tags",
  "processors": [
    {
      "uppercase": {
        "field": "message",
        "if": "ctx.tags != null && ctx.tags.contains('uppercase')"
      }
    },
    {
      "set": {
        "field": "message",
        "value": "[REDACTED]",
        "if": "ctx.tags != null && ctx.tags.contains('sensitive')"
      }
    }
  ]
}
```

**What am I seeing?**

- `if`: A Painless script condition.
- `ctx.tags != null`: Ensures `tags` exists to avoid null errors.
- `ctx.tags.contains('uppercase')`: Checks if the array includes the string "uppercase".
- Processors: 
    - `uppercase` runs if "uppercase" is in `tags`
    - `set` runs if "sensitive" is in `tags`

So this time, we're just using `set` instead of `redact`. Can you already see what's going to happen here?

**Run it**: Paste this into **Dev Tools** and execute it.

**Simulate:**

Let's simulate with the exact same data as the previous test.

```
POST _ingest/pipeline/tags-based-processing/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "secret info ssn is 123-45-6789",
        "tags": ["uppercase", "sensitive"]
      }
    }
  ]
}
```

**Output:**

```
{
  "docs": [
    {
      "doc": {
        "_source": {
          "message": "[REDACTED]",
          "tags": ["uppercase", "sensitive"]
        }
      }
    }
  ]
}
```

The order of processors matters! In this case, the `uppercase` processor ran first, but did it matter? Because "sensitive" was in `tags`, the `set` processor overwrote the entire field.

---

### Step 5: Adding your own tags

This is all well and good, you say. But what if I want to add tags based on conditions I decide?

This is where the `append` processor comes into play.

Here's a sample of the `append` processor:

```
{
  "append": {
    "field": "tags",
    "value": ["production", "{{{app}}}", "{{{owner}}}"]
  }
}
```

**What am I seeing?**

- `field`: The field to append a value to
- `value`: The value, or values to append
    - `"production"` is adding a simple string value
    - `"{{{app}}}"` is adding the value of field `app` as a tag by way of mustache templating
    - `"{{{owner}}}"` is adding the value of field `owner` as a tag by way of mustache templating

We will look deeper into mustache templating in the next part of the series, Ingest Pipelines 301.

Let's go to the next step and see how we can make use of this

---

### Step 6: Create a pipeline which conditionally adds and checks tags

```
PUT _ingest/pipeline/tags-based-processing
{
  "description": "Processes data based on tags",
  "processors": [
    {
      "redact": {
        "field": "message",
        "patterns": ["%{SSN:ssn}"],
        "pattern_definitions": {
          "SSN": "(\\d{3})(?:-(\\d{2})-|\\.(\\d{2})\\.|\\s(\\d{2})\\s)(\\d{4})"
        },
        "if": "ctx.tags != null && ctx.tags.contains('sensitive')"
      }
    },
    {
      "append": {
        "field": "tags",
        "value": "redacted",
        "if": "ctx?.message.contains('<ssn>')"
      }
    },
    {
      "uppercase": {
        "field": "message",
        "if": "ctx.tags != null && ctx.tags.contains('uppercase')"
      }
    },
    {
      "set": {
        "field": "pii_redacted",
        "value": true,
        "if": "ctx.tags != null && ctx.tags.contains('redacted')"
      }
    }
  ]
}
```

---

### Step 7: Test the updated pipeline with `_simulate`

Let's see what our results look like this time:

**Simulate:**

```
POST _ingest/pipeline/tags-based-processing/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "secret info ssn is 123-45-6789",
        "tags": ["uppercase", "sensitive"]
      }
    }
  ]
}
```

**Output:**

```
    {
      "doc": {
        "_source": {
          "pii_redacted": true,
          "message": "SECRET INFO SSN IS <SSN>",
          "tags": [
            "uppercase",
            "sensitive",
            "redacted"
          ]
        }
      }
    }
```

**What am I seeing?**

- `pii_redacted` is `true` because our condition was met and this field was added
- `message` has been converted to uppercase because our condition for that processor was met ("uppercase" in `tags`)
- "redacted" added to `tags` because our condition `ctx?.message.contains('<ssn>')` was met.

"But wait!" I hear you say. `<SSN>` is capitalized in `message`! Our condition checked for lower case `<ssn>`. How did our condition get met?

The answer is in the order of our processors. The `append` processor with the `if` statement comes before the `uppercase` processor, so that condition was true then.
