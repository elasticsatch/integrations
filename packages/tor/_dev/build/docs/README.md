# Tor Exit Node List (Unofficial) integration for Elastic

## Overview

This is an unofficial integration. It is not endorsed or sponsored by, or affiliated with, The Tor Project. Tor™ is a trademark of The Tor Project; all rights reserved. See the official Tor website at [https://torproject.org](https://torproject.org).

The Tor integration collects the IP addresses of Tor exit nodes from the [bulk exit list](https://check.torproject.org/torbulkexitlist) published by [The Tor Project](https://www.torproject.org/).

Tor exit node addresses are useful for detection and investigation, for example to flag logins, API calls, or inbound connections that originate from the Tor network. They can also be used to exempt impossible-travel false alarms when a user appears in distant locations because successive sessions egressed through different Tor exit nodes.

### Compatibility

This integration requires Kibana `^8.19.0` or `^9.1.0`, as declared in the package manifest. It works with Elastic Agent managed by Fleet and with agentless deployments where available, and it needs Elasticsearch GeoIP databases (including `GeoLite2-ASN`) for enrichment.

This integration reads the plain text list served at `https://check.torproject.org/torbulkexitlist`. The list contains one IP address per line. The list does not carry any per-address timestamp.

### How it works

Elastic Agent uses the [CEL input](https://www.elastic.co/docs/reference/beats/filebeat/filebeat-input-cel) to check the list at a configurable interval (default `1h`). The request carries the `If-Modified-Since` header with the `Last-Modified` value of the last successful download, and the list is ingested only if the server returns `200`. A `304 Not Modified` response ingests nothing. The server was observed to support conditional requests this way, but the Tor Project does not document it. Every non-empty line that does not start with `#` becomes one event. The ingest pipeline stores the address in `host.ip` and sets `host.type` to `tor_exit_node`. It also copies the address to `source.ip` and enriches `source.geo` and `source.as` (autonomous system number and organization) with the Elasticsearch GeoIP databases.

The Tor Project states that the list is updated at most once every 40 minutes, and that fetching it every hour is reasonable. It does not document a fixed update time, so do not set the interval below `40m`. Each agent fetches the list directly from The Tor Project; Elastic does not redistribute it, and no event data from your environment is sent to The Tor Project. This integration uses the time of ingestion for `@timestamp` and `event.ingested`.

## What data does this integration collect?

This integration collects the following dataset:

- `exit_node`: Tor exit node IP addresses. Documents are written to `logs-tor.exit_node-<namespace>`.

The integration also installs a [latest transform](https://www.elastic.co/docs/explore-analyze/transforms/transform-overview#latest-transform-overview) that keeps one document per address in `logs-tor_latest.dest_exit_node-1`, available through the alias `logs-tor_latest.exit_node`. An address that is no longer in the list expires from the latest index after the configured **Expiration**, counted from the last time it was ingested. The default is `2h`, slightly longer than the Tor Project's list update cadence, so the latest index converges to the current bulk list. Addresses still in the list are refreshed on every successful download. The latest index keeps only the most recent document per address, so it does not record the time ranges during which an address was an exit node.

The latest index is created with `index.mode: lookup` so that it can be used with the ES|QL [`LOOKUP JOIN`](https://www.elastic.co/docs/reference/query-languages/esql/esql-lookup-join) command, for example:

```esql
FROM logs-*
| LOOKUP JOIN logs-tor_latest.exit_node ON host.ip
```

The lookup index mode cannot be applied to a data stream, so `logs-tor.exit_node-<namespace>` itself is a regular data stream and cannot be used on the right side of a join. Use the latest index instead. A lookup index is limited to one shard.

List entries can remain for a long time after a relay stops exiting, and an exit node address is not by itself malicious. Use this data as an enrichment or risk signal rather than for automatic blocking.

### Supported use cases

- Look up whether an address seen in your events is a Tor exit node, for example with an enrich policy or a custom detection rule built on `logs-tor_latest.exit_node`.
- Hunt for activity that originates from the Tor network, including grouping exit nodes by autonomous system (`source.as`).
- Use the **[Tor Exit Node] Overview** dashboard (data view `logs-tor.exit_node-*`) for a quick view of exit-node volume, geography, and ASN.

## What do I need to use this integration?

- Elastic Agent with network access to `check.torproject.org` over HTTPS (or a proxy that can reach it).
- No credentials are required.
- The latest transform requires the permissions described in the [transform setup](https://www.elastic.co/docs/explore-analyze/transforms/transform-setup) documentation. The transform runs with the credentials of the user who installs the integration, so that user also needs read access to `logs-tor.exit_node-*` and write access to `logs-tor_latest.*`. If the user lacks these privileges, the transform is installed but deferred. Reauthorize it from the **Assets** tab of the integration with a user that has them.

## How do I deploy this integration?

### Onboard and configure

1. In Kibana, go to **Management > Integrations** and search for **Tor**.
2. Select **Add Tor**.
3. Adjust the **Interval** and **Expiration** if needed. The defaults are `1h` and `2h`.
4. Select the agent policy and save the integration.

### Validation

1. In Kibana, go to **Discover** and select the `logs-*` data view.
2. Filter with `data_stream.dataset : "tor.exit_node"`.
3. Confirm that documents contain `host.ip` and `host.type: tor_exit_node`.
4. In **Stack Management > Transforms**, confirm that the transform for `logs-tor_latest.dest_exit_node-1` is started and healthy.

## Troubleshooting

- If no documents arrive, check the Elastic Agent logs for HTTP errors. A non-200 response is reported as an event with `error.message`.
- To inspect the exact requests and responses, enable **Enable request tracing** in the integration settings. Tracing writes data to the agent's local file system, so use it only while debugging.
- If the agent must use a proxy, set **Proxy URL**.
- If `source.geo` or `source.as` is missing, confirm that the Elasticsearch GeoIP databases are available and not disabled in your cluster. Private or reserved addresses are not enriched.
- If the latest index is empty or stale, check that the transform is started and has the required permissions. See the **Assets** tab to reauthorize it.
- If no events arrive after the first download, this is expected while the list is unchanged: the server answers `304 Not Modified` and nothing is ingested until the list changes.
- An invalid **Expiration** value falls back to `2h`.

## Performance and scaling

The list is small (on the order of a thousand addresses). The full list is ingested only when it has changed since the last successful request, so the number of documents per day is the list size multiplied by the number of list updates per day. The Tor Project states that the list is updated at most once every 40 minutes. Documents in the data stream are retained for 5 days by default.

## Reference

### Tor exit node

The data is retrieved from the [Tor Project bulk exit list](https://check.torproject.org/torbulkexitlist).

#### Example event

{{event "exit_node"}}

#### ECS field reference

{{fields "exit_node"}}

### Data stream

| Data stream | Type | Description |
|---|---|---|
| `tor.exit_node` | logs | One event per Tor exit node IP address, collected with the CEL input. |

### Inputs used

{{ inputDocs }}
