# Tor Exit Node List (Unofficial) integration for Elastic

## Overview

This is an unofficial integration. It is not endorsed or sponsored by, or affiliated with, The Tor Project. Tor™ is a trademark of The Tor Project; all rights reserved. See the official Tor website at [https://torproject.org](https://torproject.org).

The Tor integration collects the IP addresses of Tor exit nodes from the [bulk exit list](https://check.torproject.org/torbulkexitlist) published by [The Tor Project](https://www.torproject.org/).

Tor exit node addresses are useful for detection and investigation, for example to flag logins, API calls, or inbound connections that originate from the Tor network.

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

An example event for `exit_node` looks as following:

```json
{
    "@timestamp": "2026-10-26T08:00:00.000Z",
    "data_stream": {
        "dataset": "tor.exit_node",
        "namespace": "ep",
        "type": "logs"
    },
    "ecs": {
        "version": "9.5.0"
    },
    "event": {
        "category": [
            "network"
        ],
        "ingested": "2026-10-26T08:00:00.000000000Z",
        "kind": "state",
        "original": "192.0.2.1",
        "type": [
            "info"
        ],
        "dataset": "tor.exit_node",
        "module": "tor"
    },
    "host": {
        "ip": [
            "192.0.2.1"
        ],
        "type": "tor_exit_node"
    },
    "input": {
        "type": "cel"
    },
    "labels": {
        "interval": "10m"
    },
    "related": {
        "ip": [
            "192.0.2.1"
        ]
    },
    "source": {
        "ip": "192.0.2.1",
        "geo": {
            "city_name": "Las Vegas",
            "continent_name": "North America",
            "country_iso_code": "US",
            "country_name": "United States",
            "location": {
                "lat": 36.17497,
                "lon": -115.13722
            },
            "region_iso_code": "US-NV",
            "region_name": "Nevada"
        },
        "as": {
            "number": 64500,
            "organization": {
                "name": "Documentation ASN"
            }
        }
    },
    "tags": [
        "preserve_original_event",
        "forwarded",
        "tor-exit-node"
    ],
    "tor": {
        "exit_node": {
            "expires_at": "2026-10-26T10:00:00.000Z"
        }
    }
}
```

#### ECS field reference

**Exported fields**

| Field | Description | Type |
|---|---|---|
| @timestamp | Date/time when the event originated. This is the date/time extracted from the event, typically representing when the event was generated by the source. If the event source has no original timestamp, this value is typically populated by the first time the event was received by the pipeline. Required field for all events. | date |
| data_stream.dataset | The field can contain anything that makes sense to signify the source of the data. Examples include `nginx.access`, `prometheus`, `endpoint` etc. For data streams that otherwise fit, but that do not have dataset set we use the value "generic" for the dataset value. `event.dataset` should have the same value as `data_stream.dataset`. Beyond the Elasticsearch data stream naming criteria noted above, the `dataset` value has additional restrictions:   \* Must not contain `-`   \* No longer than 100 characters | constant_keyword |
| data_stream.namespace | A user defined namespace. Namespaces are useful to allow grouping of data. Many users already organize their indices this way, and the data stream naming scheme now provides this best practice as a default. Many users will populate this field with `default`. If no value is used, it falls back to `default`. Beyond the Elasticsearch index naming criteria noted above, `namespace` value has the additional restrictions:   \* Must not contain `-`   \* No longer than 100 characters | constant_keyword |
| data_stream.type | An overarching type for the data stream. Currently allowed values are "logs" and "metrics". We expect to also add "traces" and "synthetics" in the near future. | constant_keyword |
| ecs.version | ECS version this event conforms to. `ecs.version` is a required field and must exist in all events. When querying across multiple indices -- which may conform to slightly different ECS versions -- this field lets integrations adjust to the schema version of the events. | keyword |
| error.code | Error code describing the error. | keyword |
| error.id | Unique identifier for the error. | keyword |
| error.message | Error message. | match_only_text |
| event.category | This is one of four ECS Categorization Fields, and indicates the second level in the ECS category hierarchy. `event.category` represents the "big buckets" of ECS categories. For example, filtering on `event.category:process` yields all events relating to process activity. This field is closely related to `event.type`, which is used as a subcategory. This field is an array. This will allow proper categorization of some events that fall in multiple categories. | keyword |
| event.dataset | Name of the dataset. If an event source publishes more than one type of log or events (e.g. access log, error log), the dataset is used to specify which one the event comes from. It's recommended but not required to start the dataset name with the module name, followed by a dot, then the dataset name. | constant_keyword |
| event.ingested | Timestamp when an event arrived in the central data store. This is different from `@timestamp`, which is when the event originally occurred.  It's also different from `event.created`, which is meant to capture the first time an agent saw the event. In normal conditions, assuming no tampering, the timestamps should chronologically look like this: `@timestamp` \< `event.created` \< `event.ingested`. | date |
| event.kind | This is one of four ECS Categorization Fields, and indicates the highest level in the ECS category hierarchy. `event.kind` gives high-level information about what type of information the event contains, without being specific to the contents of the event. For example, values of this field distinguish alert events from metric events. The value of this field can be used to inform how these kinds of events should be handled. They may warrant different retention, different access control, it may also help understand whether the data is coming in at a regular interval or not. | keyword |
| event.module | Name of the module this data is coming from. If your monitoring agent supports the concept of modules or plugins to process events of a given source (e.g. Apache logs), `event.module` should contain the name of this module. | constant_keyword |
| event.original | Raw text message of entire event. Used to demonstrate log integrity or where the full log message (before splitting it up in multiple parts) may be required, e.g. for reindex. This field is not indexed and doc_values are disabled. It cannot be searched, but it can be retrieved from `_source`. If users wish to override this and index this field, please see `Field data types` in the `Elasticsearch Reference`. | keyword |
| event.type | This is one of four ECS Categorization Fields, and indicates the third level in the ECS category hierarchy. `event.type` represents a categorization "sub-bucket" that, when used along with the `event.category` field values, enables filtering events down to a level appropriate for single visualization. This field is an array. This will allow proper categorization of some events that fall in multiple event types. | keyword |
| host.ip | Host ip addresses. | ip |
| host.type | Type of host. For Cloud providers this can be the machine type like `t2.medium`. If vm, this could be the container, for example, or other information meaningful in your environment. | keyword |
| input.type | Type of Filebeat input. | keyword |
| labels.interval | User-configured value for the `Interval` setting. | keyword |
| tor.exit_node.expires_at | The timestamp when the address expires from the latest destination index. It is `event.ingested` plus the configured expiration (`2h` by default). Addresses that remain in the bulk list are refreshed on each download; addresses that drop off expire after this grace period. | date |
| related.ip | All of the IPs seen on your event. | ip |
| source.as.number | Unique number allocated to the autonomous system. The autonomous system number (ASN) uniquely identifies each network on the Internet. | long |
| source.as.organization.name | Organization name. | keyword |
| source.as.organization.name.text | Multi-field of `source.as.organization.name`. | match_only_text |
| source.geo.city_name | City name. | keyword |
| source.geo.continent_code | Two-letter code representing continent's name. | keyword |
| source.geo.continent_name | Name of the continent. | keyword |
| source.geo.country_iso_code | Country ISO code. | keyword |
| source.geo.country_name | Country name. | keyword |
| source.geo.location | Longitude and latitude. | geo_point |
| source.geo.name | User-defined description of a location, at the level of granularity they care about. Could be the name of their data centers, the floor number, if this describes a local physical entity, city names. Not typically used in automated geolocation. | keyword |
| source.geo.postal_code | Postal code associated with the location. Values appropriate for this field may also be known as a postcode or ZIP code and will vary widely from country to country. | keyword |
| source.geo.region_iso_code | Region ISO code. | keyword |
| source.geo.region_name | Region name. | keyword |
| source.geo.timezone | The time zone of the location, such as IANA time zone name. | keyword |
| source.ip | IP address of the source (IPv4 or IPv6). | ip |
| tags | List of keywords used to tag each event. | keyword |


### Data stream

| Data stream | Type | Description |
|---|---|---|
| `tor.exit_node` | logs | One event per Tor exit node IP address, collected with the CEL input. |

### Inputs used

These inputs can be used with this integration:
<details>
<summary>cel</summary>

## Setup

For more details about the CEL input settings, check the [Filebeat documentation](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-cel.html).

Before configuring the CEL input, make sure you have:
- Network connectivity to the target API endpoint
- Valid authentication credentials (API keys, tokens, or certificates as required)
- Appropriate permissions to read from the target data source

### Collecting logs from CEL

To configure the CEL input, you must specify the `request.url` value pointing to the API endpoint. The interval parameter controls how frequently requests are made and is the primary way to balance data freshness with API rate limits and costs. Authentication is often configured through the `request.headers` section using the appropriate method for the service.

NOTE: To access the API service, make sure you have the necessary API credentials and that the Filebeat instance can reach the endpoint URL. Some services may require IP whitelisting or VPN access.

To collect logs via API endpoint, configure the following parameters:

- API Endpoint URL
- API credentials (tokens, keys, or username/password)
- Request interval (how often to fetch data)
</details>

