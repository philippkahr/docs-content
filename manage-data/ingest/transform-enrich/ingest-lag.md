---
mapped_pages:
  - https://www.elastic.co/docs/manage-data/ingest/transform-enrich/calculate-ingest-lag.html
applies_to:
  stack: ga
  serverless: ga
---

# Ingest lag

This is a recurring topic and theme and deserves its own heading. The idea is always the same, calculate the time it takes from reading a document, until you receive it in Elasticsearch. Store that in minutes, seconds, milliseconds, and use it to quickly graph and alert on.

The generally working idea is the following: `event.ingested - @timestamp`

The `event.ingested` is available in two ways:

- `_ingest.timestamp`

Available using mustache notation `{{_ingest.timestamp}}` in all processors except script.

- `metadata().now`

Available only in the script processor. Use it instead of `_ingest.timestamp`

The `event.ingested` will be set in the `fleet final pipeline` but that one runs at the latest possible time. The calculation in `SECONDS` is usually granular enough.

The following script is the main magic and the minimum you should implement. It will create a new field called `event.ingestion.latency` and you can leverage that to see the full progress.

```json
{
  "script": {
    "description": "Calculates entire ingestion flow latency",
    "if": "ctx['@timestamp'] != null",
    "source": """
      ZonedDateTime start = ZonedDateTime.parse(ctx['@timestamp']);
      ctx.putIfAbsent("event", [:]);
      ctx.event.putIfAbsent("ingestion", [:]);
      ctx.event.ingestion.latency= ChronoUnit.SECONDS.between(start, metadata().now);
    """
  }
}
```

## @timestamp

One key important aspect is that `@timestamp` can either be the document when Elastic Agent has read the document, or the real timestamp out of the document, after it has been parsed. The main culprit here is that this is not always the same, because when Elastic Agent reads Windows event logs, it will already set the @`timestamp` based on the log. It just doesn’t do it for everything, e.g. syslog, or reading Linux log files.

Here is an example of how this can be good and bad at the same time.

```json
POST _ingest/pipeline/_simulate
{
    "docs": [{
        "_source": {
            "@timestamp": "2025-04-03T10:00:00.000Z",
            "message": "2025-03-01T09:00:00.000Z user: philipp has logged in"
        }
    }],
    "pipeline": {
        "processors": [
          {"script": {
            "if": "ctx['@timestamp'] != null",
            "source": """
              ZonedDateTime start = ZonedDateTime.parse(ctx['@timestamp']);
              ctx.latency= ChronoUnit.SECONDS.between(start, metadata().now);
                """
          }}
        ]
    }
}
```

In the following example above, we can tell that the timestamp, this is when Elastic Agent read the document is `3rd April at 10:00` and the log message on the disk is from 3rd March. When we do the `diff` at the very first step, before any parsing we can be sure that this will be proper. When we execute our calculation as the last step in the pipeline (usually what is happening with Elastic Integrations leveraging `@custom` pipelines), then the date `2025-03-01` becomes the `@timestamp` and the latency calculation will be erroneous.

We can’t always solve all the issues, sometimes we get to a good enough situation with the above and simply saying `@timestamp` is good, since we expect Elastic Agent to pick up the log as fast as possible anyway. In the beginning and onboarding process of new data sources, the latency might be higher, since we might read in old data.

## Architectures

Regardless of the chosen architecture it is a good idea to add a `remove` processor at the end that drops the `_tmp` field. You don’t need the raw timestamps from the various hops, the latency in seconds should be enough.

### Elastic Agent \=\> Elasticsearch

We can use `@timestamp` and `event.ingested` and calculate the difference. This will give you the following document. The `event.ingestion.latency` is in seconds.

```json
{
  "event": {
    "ingestion": {
      "latency": 443394
    }
  }
}
```

#### Script

```json
POST _ingest/pipeline/_simulate
{
    "docs": [{
        "_source": {
            "@timestamp": "2025-04-03T10:00:00.000Z",
            "message": "user: philipp has logged in",
            "_tmp": {
              "logstash": "2025-04-03T10:00:02.456Z"
            }

        }
    }],
    "pipeline": {
        "processors": [
          {
            "script": {
              "description": "Calculates entire ingestion flow latency",
              "if": "ctx['@timestamp'] != null",
              "source": """
                ZonedDateTime start = ZonedDateTime.parse(ctx['@timestamp']);
                ctx.putIfAbsent("event", [:]);
                ctx.event.putIfAbsent("ingestion", [:]);
                ctx.event.ingestion.latency= ChronoUnit.SECONDS.between(start, metadata().now);
              """
            }
          }
        ]
    }
}
```

### Elastic Agent \=\> Logstash \=\> Elasticsearch

In this case we have an additional hop that we need to manage. We know that Elastic Agent populates the `@timestamp`. Logstash does not add any timestamp per default, we would recommend adding a temporary timestamp (simply adding it to `_tmp.logstash_seen`). You can calculate the following values:

- Total latency (@timestamp \- event.ingested)
- Elastic Agent \=\> Logstash (@timestamp \- \_tmp.logstash_seen)
- Logstash \=\> Elasticsearch (\_tmp.logstash_seen \- event.ingested)

Those values can be useful in any debugging scenario, as you can quickly tell where the lag is created. Is Elastic Agent to Logstash slow, or is Logstash to Elasticsearch slow.

Below is a script that will generate the following values. This will calculate the difference for all the different parts as explained above.

```json
{
  "event": {
    "ingestion": {
        "latency_logstash_to_elasticsearch": 443091,
        "latency": 443093,
        "latency_elastic_agent_to_logstash": 1
      }
  }
}
```

#### Script

If you want to remove the first calculation, you will need to ensure that the object `event.ingestion` is available.

```json
POST _ingest/pipeline/_simulate
{
    "docs": [{
        "_source": {
            "@timestamp": "2025-04-03T10:00:00.000Z",
            "message": "user: philipp has logged in",
            "_tmp": {
              "logstash": "2025-04-03T10:00:02.456Z"
            }

        }
    }],
    "pipeline": {
        "processors": [
          {
            "script": {
              "description": "Calculates entire ingestion flow latency",
              "if": "ctx['@timestamp'] != null",
              "source": """
                ZonedDateTime start = ZonedDateTime.parse(ctx['@timestamp']);
                ctx.putIfAbsent("event", [:]);
                ctx.event.putIfAbsent("ingestion", [:]);
                ctx.event.ingestion.latency= ChronoUnit.SECONDS.between(start, metadata().now);
              """
            }
          },
          {
            "script": {
              "description": "Calculates logstash to Elasticsearch latency",
              "if": "ctx._tmp?.logstash != null",
              "source": """
                ZonedDateTime start = ZonedDateTime.parse(ctx._tmp.logstash);
                ctx.event.ingestion.latency_logstash_to_elasticsearch=ChronoUnit.SECONDS.between(start, metadata().now);
              """
            }
          },
          {
            "script": {
              "description": "Calculates Elastic Agent to Logstash latency",
              "if": "ctx._tmp?.logstash != null",
              "source": """
                ZonedDateTime start = ZonedDateTime.parse(ctx['@timestamp']);
                ZonedDateTime end = ZonedDateTime.parse(ctx._tmp.logstash);
                ctx.event.ingestion.latency_elastic_agent_to_logstash=ChronoUnit.SECONDS.between(start, end);
              """
            }
          }
        ]
    }
}
```

### Elastic Agent \=\> Logstash \=\> Kafka \=\> Logstash \=\> Elasticsearch

Similar to the story above, we add an additional hop and therefore the recommendation is to add an additional temporary timestamp field. Please read the above `Elastic Agent => Logstash => Elasticsearch` heading for more insights.

Below is a script that will generate the following values. This will calculate the difference for all the different steps.

```json
{
  "event": {
    "ingestion": {
            "latency_logstash_to_elasticsearch": 443091,
            "latency_logstash_to_logstash": 1,
            "latency": 443093,
            "latency_elastic_agent_to_logstash": 1
      }
  }
}
```

#### Script

If you want to remove the first calculation, you will need to ensure that the object `event.ingestion` is available. Of course you could merge all of the steps into one larger script. I personally like to separate it, so you can edit, modify and enhance exactly what you need.

```json
POST _ingest/pipeline/_simulate
{
    "docs": [{
        "_source": {
            "@timestamp": "2025-04-03T10:00:00.000Z",
            "message": "user: philipp has logged in",
            "_tmp": {
              "logstash_pre_kafka": "2025-04-03T10:00:01.233Z",
              "logstash_post_kafka": "2025-04-03T10:00:02.456Z"
            }

        }
    }],
    "pipeline": {
        "processors": [
          {
            "script": {
              "description": "Calculates entire ingestion flow latency",
              "if": "ctx['@timestamp'] != null",
              "source": """
                ZonedDateTime start = ZonedDateTime.parse(ctx['@timestamp']);
                ctx.putIfAbsent("event", [:]);
                ctx.event.putIfAbsent("ingestion", [:]);
                ctx.event.ingestion.latency= ChronoUnit.SECONDS.between(start, metadata().now);
              """
            }
          },
          {
            "script": {
              "description": "Calculates logstash to logstash latency",
              "if": "ctx._tmp?.logstash_pre_kafka != null && ctx._tmp?.logstash_post_kafka != null",
              "source": """
                ZonedDateTime start = ZonedDateTime.parse(ctx._tmp.logstash_pre_kafka);
                ZonedDateTime end = ZonedDateTime.parse(ctx._tmp.logstash_post_kafka);
                ctx.event.ingestion.latency_logstash_to_logstash=ChronoUnit.SECONDS.between(start, end);
              """
            }
          },
          {
            "script": {
              "description": "Calculates logstash post Kafka to Elasticsearch latency",
              "if": "ctx._tmp?.logstash_post_kafka != null",
              "source": """
                ZonedDateTime start = ZonedDateTime.parse(ctx._tmp.logstash_post_kafka);
                ctx.event.ingestion.latency_logstash_to_elasticsearch=ChronoUnit.SECONDS.between(start, metadata().now);
              """
            }
          },
          {
            "script": {
              "description": "Calculates Elastic Agent to pre kafka Logstash latency",
              "if": "ctx._tmp?.logstash_pre_kafka != null",
              "source": """
                ZonedDateTime start = ZonedDateTime.parse(ctx['@timestamp']);
                ZonedDateTime end = ZonedDateTime.parse(ctx._tmp.logstash_pre_kafka);
                ctx.event.ingestion.latency_elastic_agent_to_logstash=ChronoUnit.SECONDS.between(start, end);
              """
            }
          }
        ]
    }
}
```
