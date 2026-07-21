---
layout: post
title: "Understanding Grafana Loki: Labels, Streams, and Chunks"
tags: [Observability, Grafana Loki, Logging]
image: /assets/images/loki/loki-labels-streams-chunks-cover.png
---

This is the first post in a series where I am studying Grafana Loki's architecture from the ground up: why it exists, how logs move through the system, how the ring distributes work, how storage is organized, and how queries run without indexing every log line.

The first idea that changes how I think about Loki is also the easiest one to misunderstand: Loki does not build a full-text index of every log line.

That sounds limiting at first. Logs are text, so it feels natural to index the text. But Loki is built around a different tradeoff. It indexes labels, stores log lines in compressed chunks, and scans those chunks at query time after the labels have narrowed the search space.

The core rule is:

> Use labels to find the right streams; scan chunks to find the matching lines.

This is the same reason Loki often feels closer to Prometheus than to a traditional search engine. The index is not a giant map from every word to every log line. It is a smaller structure that helps Loki answer a different question first: which streams could contain the logs I care about?

## The logging problem at scale

Logs grow quickly because they are produced by almost everything:

- Every service instance
- Every request path
- Every background job
- Every dependency call
- Every retry, timeout, warning, and error

At small scale, indexing all of that text can feel convenient. You send logs to a search system, type a word, and expect matching lines to appear.

At larger scale, the convenience becomes expensive. A full-text log index has to process, tokenize, index, and retain a searchable representation of the log contents. That means more CPU during ingestion, more storage for the index, more background compaction, and more operational work to keep the cluster healthy.

The difficult part is that a lot of log data is rarely queried. Many logs are useful because they exist when something goes wrong, not because every word in them needs to be indexed forever.

Loki starts from that observation. Instead of treating logs as documents that must be fully indexed, it treats them as streams that can be selected by metadata.

## Why Elasticsearch is not always the right fit

Elasticsearch is powerful when the main requirement is flexible search over document contents. If I need rich text search, relevance scoring, complex inverted indexes, and exploratory queries across arbitrary fields, that model makes sense.

But operational logs often have a different access pattern.

Usually I start with context:

- Which application?
- Which namespace?
- Which environment?
- Which cluster?
- Which container?
- What time range?

Only after that do I search for a specific word, error message, trace ID, or JSON field.

In other words, I rarely need to ask, "where did this word appear across every log ever stored?" More often I ask, "inside this service, during this time window, which lines mention this failure?"

Loki is optimized for that second question. It avoids the cost of indexing every log line by relying on labels to reduce the amount of data that must be scanned.

## Labels first

The most important Loki concept is the label set. A label is a key-value pair that describes where a log came from or what stable context it belongs to.

Examples:

```text
app="checkout"
namespace="production"
cluster="us-east-1"
container="api"
```

A Loki query begins with a label selector:

```logql
{app="checkout", namespace="production"}
```

That selector does not search the log text. It selects streams whose labels match the query. After Loki has found candidate streams and chunks for the requested time range, the rest of the LogQL pipeline can filter the actual log lines:

```logql
{app="checkout", namespace="production"} |= "payment failed"
```

The label selector is the narrowing step. The text filter is the scanning step.

That distinction is the center of Loki's design.

## Streams

A stream is the sequence of log entries that share the exact same label set.

These two entries belong to the same stream if their labels are identical:

```text
{app="checkout", namespace="production", container="api"}
```

If one label value changes, Loki sees a different stream:

```text
{app="checkout", namespace="production", container="worker"}
```

This is useful because logs from the same source usually arrive together and are queried together. Loki can group them into chunks and index the metadata for those chunks instead of indexing every individual line.

The stream model also explains why labels must be chosen carefully. Labels define how many streams Loki has to manage.

## Chunks

Loki stores the actual log lines in chunks. A chunk contains log entries for a stream over a time range. It is compressed and eventually flushed to storage.

A simplified way to see it is:

```text
stream labels
    -> chunk from 10:00 to 10:15
    -> chunk from 10:15 to 10:30
    -> chunk from 10:30 to 10:45
```

The index can tell Loki which chunks might matter for a query. The chunk contains the real log content.

This is different from a full-text search system. A full-text index tries to answer "which documents contain this term?" directly from the index. Loki's index answers "which chunks belong to streams matching these labels and this time range?" Then Loki reads and filters those chunks.

The benefit is that the index can stay much smaller. The cost is that queries may need to scan chunk data, especially if the label selector is broad.

## Object storage

Loki is designed to use object storage as the long-term place for chunks and, in modern single-store setups, index data too. The Grafana documentation describes Loki storage around a small index plus compressed chunks in object stores such as S3.

This matters because object storage changes the cost model. Instead of keeping a large search index on expensive local disks across many indexing nodes, Loki can store compressed log data in a cheaper backend and scale the read path when queries need to inspect it.

That is one reason the design is attractive for large volumes of logs. Loki does not make querying free, but it tries to make storing logs less expensive and simpler to operate.

## Cardinality is the trap

The label model only works if labels stay low-cardinality.

Cardinality means the number of distinct values a label can have. A label like `environment` is usually low-cardinality:

```text
environment="dev"
environment="staging"
environment="production"
```

A label like `trace_id` is high-cardinality:

```text
trace_id="0242ac120002"
trace_id="ae91bc78f1d4"
trace_id="f0c41e9b77aa"
```

Every unique label set creates a separate stream. If a value changes for every request, then Loki has to manage a huge number of short-lived streams. That creates a bigger index, more tiny chunks, more flushing, and worse query behavior.

Good labels usually describe stable origin or routing context:

- `app`
- `namespace`
- `cluster`
- `environment`
- `container`

Bad labels usually describe individual events:

- `request_id`
- `trace_id`
- `user_id`
- `order_id`
- `timestamp`
- Raw URL paths with unbounded IDs

This is the part of Loki that feels simple but has deep operational consequences. Loki is not saying metadata does not matter. It is saying that indexed labels should be metadata with controlled cardinality.

## Structured metadata

The uncomfortable question is: what should I do with fields that are useful but too high-cardinality for labels?

For example, trace IDs are extremely useful when connecting logs to traces. But using `trace_id` as a label would create many streams, often one per request.

Structured metadata is Loki's answer for some of this middle ground. It lets metadata travel with the log entry without becoming part of the indexed label set. The data can still be used in queries, but it does not expand the index the same way a label would.

A useful mental model is:

- Labels decide where Loki should look first.
- Structured metadata gives extra context after Loki has found candidate log entries.
- Log line filters inspect the text itself.

I will explore more about it in a future post.

## Why this design works

Loki's architecture makes more sense when I separate storage cost from query cost.

Full-text indexing pays more during ingestion and storage so arbitrary text searches can be faster later. Loki pays less during ingestion and storage by keeping a smaller index, then accepts that some queries will scan compressed chunks.

That is a good tradeoff when:

- Logs are very large in volume.
- Most queries start with service, namespace, environment, or cluster.
- The team can control label cardinality.
- Object storage is the desired long-term backend.
- The system is used for observability workflows, not general document search.

It is a worse tradeoff when:

- Users expect arbitrary full-text search across everything.
- Label selection is too broad.
- High-cardinality values are accidentally promoted to labels.
- The system produces many short-lived streams.
- Queries frequently need needle-in-a-haystack searches over huge time ranges.

Loki is not magic. It is a set of constraints. When the constraints match the workload, the design is elegant. When they are ignored, the same design can become painful.

## The mental model

The best simplified model I have found is this:

```text
labels + time range
    -> find streams
    -> find chunk references
    -> read compressed chunks
    -> filter log lines
    -> return matches
```

The index is there to avoid reading everything. It is not there to answer every possible text question by itself.

This is why Loki's label design matters so much. Labels are the map. Chunks are the archive. LogQL is the tool that narrows, scans, parses, and aggregates.

## Final thought

Loki does not index log contents because it is choosing a different center of gravity. It assumes that logs are usually searched with context first and text second.

That choice gives Loki its shape: labels, streams, chunks, object storage, and query-time scanning. Once that tradeoff is clear, the rest of the architecture starts to feel less surprising.

The next question is what happens when a log actually arrives. That path goes through distributors, the ring, replication, ingesters, the WAL, chunks, and finally object storage.

## References

- [Grafana Loki documentation: Get started](https://grafana.com/docs/loki/latest/get-started/)
- [Grafana Loki documentation: Storage](https://grafana.com/docs/loki/latest/configure/storage/)
- [Grafana Loki documentation: Cardinality](https://grafana.com/docs/loki/latest/get-started/labels/cardinality/)
- [Grafana Loki documentation: Structured metadata](https://grafana.com/docs/loki/latest/get-started/labels/structured-metadata/)
