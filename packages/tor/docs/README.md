# Tor integration for Elastic

## Overview

The Tor integration collects the IP addresses of Tor exit nodes from the [bulk exit list](https://check.torproject.org/torbulkexitlist) published by [The Tor Project](https://www.torproject.org/).

Tor exit node addresses are useful for detection and investigation, for example to flag logins, API calls, or inbound connections that originate from the Tor network.

### Compatibility

This integration reads the plain text list served at `https://check.torproject.org/torbulkexitlist`. The list contains one IP address per line. The list does not carry any per-address timestamp.

### How it works

Elastic Agent uses the [CEL input](https://www.elastic.co/docs/reference/beats/filebeat/filebeat-input-cel) to request the list at a configurable interval. Every non-empty line that does not start with `#` becomes one event. The ingest pipeline stores the address in `host.ip` and sets `host.type` to `tor_exit_node`.

The Tor Project states that the list is updated at most once every 40 minutes, and that fetching it every hour is reasonable. It does not document a fixed update time, so this integration uses the time of ingestion for `@timestamp` and `event.ingested`.

## What data does this integration collect?

This integration collects the following dataset:

- `exit_node`: Tor exit node IP addresses. Documents are written to `logs-tor.exit_node-<namespace>`.

The integration also installs a [latest transform](https://www.elastic.co/docs/explore-analyze/transforms/transform-overview#latest-transform-overview) that keeps one document per address in `logs-tor_latest.dest_exit_node-1`, available through the alias `logs-tor_latest.exit_node`. An address that is no longer in the list expires from the latest index after two intervals.

The latest index is created with `index.mode: lookup` so that it can be used with the ES|QL [`LOOKUP JOIN`](https://www.elastic.co/docs/reference/query-languages/esql/esql-lookup-join) command, for example:

```esql
FROM logs-*
| LOOKUP JOIN logs-tor_latest.exit_node ON host.ip
```

The lookup index mode cannot be applied to a data stream, so `logs-tor.exit_node-<namespace>` itself is a regular data stream and cannot be used on the right side of a join. Use the latest index instead. A lookup index is limited to one shard.

### Supported use cases

- Look up whether an address seen in your events is a Tor exit node, for example with an enrich policy or a custom detection rule built on `logs-tor_latest.exit_node`.
- Hunt for activity that originates from the Tor network.

## What do I need to use this integration?

- Elastic Agent with network access to `check.torproject.org` over HTTPS (or a proxy that can reach it).
- No credentials are required.
- The latest transform requires the permissions described in the [transform setup](https://www.elastic.co/docs/explore-analyze/transforms/transform-setup) documentation.

## How do I deploy this integration?

### Onboard and configure

1. In Kibana, go to **Management > Integrations** and search for **Tor**.
2. Select **Add Tor**.
3. Adjust the **Interval** if needed. The default is `1h`.
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

## Performance and scaling

The list is small (on the order of a thousand addresses). Every request ingests the full list again, so the number of documents per day is the list size multiplied by the number of requests per day. Increase the **Interval** to reduce duplicate documents. Documents in the data stream are retained for 5 days by default.

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
        "interval": "1h"
    },
    "related": {
        "ip": [
            "192.0.2.1"
        ]
    },
    "tags": [
        "preserve_original_event",
        "forwarded",
        "tor-exit-node"
    ],
    "tor": {
        "exit_node": {
            "deleted_at": "2026-10-26T10:00:00Z"
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
| labels.interval | User-configured value for the `Interval` setting. It is used to calculate the expiration time. | keyword |
| related.ip | All of the IPs seen on your event. | ip |
| tags | List of keywords used to tag each event. | keyword |
| tor.exit_node.deleted_at | The timestamp when the address is (will be) removed from the latest destination index. It is `event.ingested` plus two times `labels.interval`. | date |


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

