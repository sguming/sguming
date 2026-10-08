# Reliable log ingestion: checkpoints, event IDs, and alert boundaries

[Back to profile](../README.md)

This note comes from a local Docker learning lab using Elasticsearch, Logstash, and Kibana 7.17.29. The checks below were recorded on **8 October 2026, Australia/Adelaide**. They describe that lab run.

## The problem

A source file contained **234 valid events**, including **24 HTTP 404 responses**, plus one malformed line. Elasticsearch held **468 documents**, while the number of unique event IDs was still 234. The ingestion pipeline had replayed the same input.

The file input used `/dev/null` for its checkpoint file, and the Elasticsearch output did not assign a stable document ID. Recreating the container could therefore reread old lines and store them as new documents.

## Two controls with different jobs

**Persist the read position.** Logstash's file input uses `sincedb` to track how far it has read. Keeping that checkpoint on persistent storage lets the input continue from its previous position after a restart. The checkpoint path must be writable and survive container recreation. [Logstash file input documentation](https://www.elastic.co/guide/en/logstash/7.17/plugins-inputs-file.html)

**Give each event a stable identity.** Use the parsed `event_id` as the Elasticsearch document ID. If an event is replayed into the same target index, it addresses the existing document. This complements the checkpoint; it does not replace it.

The relevant settings, with generic names:

```text
# In the existing file input; /data is persistent storage.
sincedb_path => "/data/.sincedb-events"

# In each Elasticsearch output, after parsing and validating event_id.
document_id => "%{event_id}"
```

These are configuration excerpts, not a complete pipeline. The lab routes lines that fail parsing into a quarantine file and applies the stable ID to both its main event index and its filtered authentication-failure index.

Stable IDs require a reliable, unique event identifier. An ID reused for different events can overwrite data, and an ID only deduplicates within its target index. File rotation and checkpoint expiry also need separate consideration in a broader deployment.

## What I checked

| Check | Recorded result |
| :--- | :--- |
| Source versus repaired main index | 234 valid events / 234 indexed events |
| Event identity | 234 unique event IDs; indexed IDs matched source IDs |
| HTTP 404 count | 24 |
| Malformed input | One line routed to quarantine |
| Container recreation | Counts stayed at 234 events and 24 HTTP 404s |
| Input after recreation | Zero old events reread |

The result was checked against the source file, document IDs, and HTTP-status counts. A single total count would have missed part of the original inconsistency.

## Test the alert on both sides of the threshold

The Kibana rule counted failed logins **per source IP over 15 minutes**, with a strict **count > 5** condition and an evaluation every minute. Kibana's index-threshold rule supports a time window, aggregation, grouping, and comparison threshold. [Kibana index-threshold documentation](https://www.elastic.co/guide/en/kibana/7.17/rule-type-index-threshold.html)

| Input within the window | Recorded result after rule execution |
| :--- | :--- |
| Five failures from one test IP | No active alert |
| Sixth failure from the same IP | Active alert: threshold met |

I kept the ingestion baseline in a fixed time window because adding alert-test events changed the all-time totals. The alert used a rolling window, so its later recovery would not invalidate the recorded threshold test.

The [recorded verification summary](./evidence/elastic-lab-summary.json) retains the relevant counts, restart result, and alert outcomes. It is a compact export of the saved local verification records; it does not reproduce the original dataset or run a live check.

The useful lesson: verify what was read, how events are identified, what survives a restart, and exactly where an alert changes state.
