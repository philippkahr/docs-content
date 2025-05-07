---
mapped_pages:
  - https://www.elastic.co/docs/manage-data/ingest/transform-enrich/error-handling.html
applies_to:
  stack: ga
  serverless: ga
---

# Error handling

**Important**: Ingest pipelines are executed before the document is indexed by Elasticsearch. You can handle the errors occurring while processing the document (ie transforming the json object) but not the errors triggered while indexing like mapping conflict, incorrect right in the index. **Recommendation** create a `error-handling-pipeline` which sets the `event.kind` to `pipeline_error` and puts the error with the tag from the processor into the `error.message` field. A tag is very useful especially if you have multiple grok, dissects, script processors and you cannot identify where it broke.

The `on_failure` can be set on a per processor or a per pipeline basis to catch the exceptions that could be raised during the processing of the document in the pipeline and its processors. The `ignore_failure` allow to not take into account errors in a specific processor.

## Global vs processor specific

Here is an example pipeline that leverages the `on_failure` handler on the pipeline level and not directly at the processor. This has a few caveats as it ensure that the pipeline exists safely, but it will not process further. In our case somebody made a typo when configuring the `dissect` processor to extract the `user.name` out of the message. Instead of a `:` it is a `,`.

```json
POST _ingest/pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "@timestamp": "2025-04-03T10:00:00.000Z",
        "message": "user: philipp has logged in"
      }
    }
  ],
  "pipeline": {
    "processors": [
      {
        "dissect": {
          "field": "message",
          "pattern": "%{}, %{user.name} %{}",
          "tag": "dissect for user.name"
        }
      },
      {
        "set": {
          "field": "event.category",
          "value": "authentication"
        }
      }
    ],
    "on_failure": [
      {
        "set": {
          "field": "event.kind",
          "value": "pipeline_error"
        }
      },
      {
        "set": {
          "field": "error.message",
          "value": "Processor {{ _ingest.on_failure_processor_type }} with tag {{ _ingest.on_failure_processor_tag }} in pipeline {{ _ingest.on_failure_pipeline }} failed with message: {{ _ingest.on_failure_message }}"
        }
      }
    ]
  }
}
```

The second processor with the `event.category` setting to `authentication` is not executed anymore, as the first dissect errors and the global `on_failure` is triggered. The document is now this. We can see the processor that errored, what pattern was tried and what the passed argument was.

```json
"@timestamp": "2025-04-03T10:00:00.000Z",
"message": "user: philipp has logged in",
"event": {
    "kind": "pipeline_error"
},
"error": {
  "message": "Processor dissect with tag dissect for user.name in pipeline _simulate_pipeline failed with message: Unable to find match for dissect pattern: %{}, %{user.name} %{} against source: user: philipp has logged in"
}
```

We can restructure this and include the same values on the processor itself, which results in the document being calculated and the `event.category` being executed. We have the global setting, for when any other subsequent processor errors and at the same time the processor specific. (The execution of the two set operators in the dissect might not make the most sense). For the dissect you might want to add a specific `_tmp.error: dissect_failure` and leveraging `if` conditions further down you can execute certain processors only when the parsing failed.

```json
POST _ingest/pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "@timestamp": "2025-04-03T10:00:00.000Z",
        "message": "user: philipp has logged in"
      }
    }
  ],
  "pipeline": {
    "processors": [
      {
        "dissect": {
          "field": "message",
          "pattern": "%{}, %{user.name} %{}",
          "on_failure": [
            {
              "set": {
                "field": "event.kind",
                "value": "pipeline_error"
              }
            },
            {
              "set": {
                "field": "error.message",
                "value": "Processor {{ _ingest.on_failure_processor_type }} with tag {{ _ingest.on_failure_processor_tag }} in pipeline {{ _ingest.on_failure_pipeline }} failed with message: {{ _ingest.on_failure_message }}"
              }
            }
          ],
          "tag": "dissect for user.name"
        }
      },
      {
        "set": {
          "field": "event.category",
          "value": "authentication"
        }
      }
    ],
    "on_failure": [
      {
        "set": {
          "field": "event.kind",
          "value": "pipeline_error"
        }
      },
      {
        "set": {
          "field": "error.message",
          "value": "Processor {{ _ingest.on_failure_processor_type }} with tag {{ _ingest.on_failure_processor_tag }} in pipeline {{ _ingest.on_failure_pipeline }} failed with message: {{ _ingest.on_failure_message }}"
        }
      }
    ]
  }
}
```
