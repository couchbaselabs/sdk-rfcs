# Meta

* RFC Name: Application Service Level Telemetry
* RFC ID: 84
* Start-Date: 2024-01-04
* Owners: Sergey Avseyev
* Contributors: Matt Ingenthron, Sergey Avseyev
* Current Status: ACCEPTED

# Summary

While Capella is well instrumented for what runs in the VPC for the user, we do
not today have much detail on how a user's deployment is operating.  This
proposal improves that by adding simple but powerful telemetry tracking from
the user's environment to Capella.

# Motivation

Couchbase Capella uplevels the user experience over a traditional Customer
Managed environment in that *As a Service* not only basic health but even
optimization and proactive checking for problems with the service level.  A
challenge here is that some of the information about service health is not
immediately available from within the Capella Data Plane; it is in the user's
application deployment in the cloud.  That said, some users have not defined
service levels and are not measuring them, but rather just respond when they
receive complaints or perceive that service levels are outside normal operating
parameters.

# General Design

This feature likely will not, in many cases, point directly to the cause of an
issue.  In most cases, it will help to ensure availability of the metrics,
particularly when other the application does to retain logs for a long enough
period.

To provide the most value at the lowest cost, two levels of metrics are
proposed:

* Capella Service Experience Counters
* Capella Service Level Metrics

## Capella Service Experience Counters

As initial background to those who are not familiar with the [SDK-RFC on error
handling](rfc/0058-error-handling.md), Couchbase SDKs attempt to consolidate
much of the error handling for an app developer into a few specific errors many
of which may have an error context. The SDKs try to balance this Couchbase
design of error handling with respecting platform idioms.

That means in SDK instances, we largely have two major errors that a user may
see along with additional error context.  The error context tends to be service
specific and these may not have a similar shape as the services were
independently developed (among other reasons).  There are a number of other
errors, like `InvalidArgument` and `KeyNotFound` which are not useful from a
Capella Experience perspective.

The two metrics we should collect are:

* Timeout.
  The requested operation has timed out.  For example, it was written to the
  network and timed out before the response was received. The SDK might still
  receive response from the Server, which is then qualified as an orphan
  response, but the Timeout counter must be incremented anyway.
* Request Canceled.
  The requested operation was started and dispatched to the intended system.
  However, it has been canceled by the SDK's processing.  Typically this is if
  the connection is severed, but it could also have other causes (e.g.,
  shutdown) and can even be triggered by the user themselves (e.g., calling
  `.cancel()` on a result from another thread)

An additional metric for total requests completed would allow to account for
other types of errors.

For this set of metrics, we propose gathering simple counters, and on a per SDK
instance basis (these do not correlate 1:1 to hostnames and a definition of
runtime/instance identifiers is in the [Response Time Observability
RFC](rfc/0035-rto.md)). Each counter must be associated with the node hostname, node UUID and
bucket if it is possible. In particular, it must be possible to label counters
with bucket name and hostname for KV service, while often not possible to even
reliably associate hostname with Search request due to proxies on the Cloud
deployments.

We should note that existence of timeouts is possibly expected.  For example,
with an ad-targeting or recommendation targeting deployment, there may be short
latencies that when not serviced, lead to other processing.  This is worth
mentioning because it will be important to compare a deployment to itself, and
there may be limited value in comparing across deployments.

## Capella Service Level Metrics

The counters described above give a good picture of actual impact on a deployed
app, but they do not give a sense of whether or not a given deployment is
meeting an expected service level.  Said another way, a Couchbase Capella
Cluster deployment could be operational, but very slow during a rebalance
leading to user discontent, but never turn into an error.

Service levels are specific to the user's targets.  While it's impossible to
properly measure whether or not an end-user's app is meeting service level
targets, we can establish a predefined histogram that would allow Couchbase to
simply and inexpensively understand if the service level has likely changed.

It's proposed that we establish a set of standardized histograms, specific to
services and to the service level expected with the service using those
parameters.  Counters for these histograms can be sent to Couchbase and both
stored individually and aggregated. See histogram buckets in "Histograms" table
below.

As an aggregated set of metrics, it would give Couchbase a rough view over the
service level the user is experiencing.  Deviations in service level can be dug
into and, in particular, we can check for correlation to Couchbase Capella
management activities like upgrades.

The Indexing Service is not included as it is not used by SDK, even for
management operations. All future services must be reported if the service
exposes separate port and the SDK is using it.

As with the Service Experience Counters, it may be incorrect to try to
compare between deployments.  For example, the query complexity may differ
significantly or the data size and use of features like compression may vary
significantly between two applications.

The SDK must record latency for histogram on the lower layer for each
transmission attempt instead of top-level operation. For example, if KV GET
operation results in three network requests, there should be three data points
for the metrics counter, and if the operation has not been canceled, it should
record three data points for histogram, even if the operation has been timed
out, because on the lower-level API, the SDK still sees orphan responses and is
able to figure out true latency of the request.

# API and Implementation Details

The decision to reuse the existing Meter implementation is considered an SDK
implementation detail, and this RFC does not specify the API for collecting
metrics from the library.

One possible approach is to introduce a "compound meter" that aggregates an
existing meter (which could be overridden by the user) and the application
service-level telemetry meter described in this RFC. This compound meter would
collect counters and fixed-bucket latency histograms. The application
service-level meter would continue to function even if the user provides a
custom meter implementation.

In addition to latencies, which would be used for reporting histograms, the
default Meter should also maintain the following counters:

| Metric | Description |
| :---- | :---- |
| sdk\_*{service}*\_r\_timedout | Timeout |
| sdk\_*{service}*\_r\_canceled | Canceled |
| sdk\_*{service}*\_r\_total | Total number of operations |

The SDK should not send zero metric.

The SDK must send histogram buckets even if it has zero hits.

Where the service is a part of the visible counter name, and hostname and
bucket must be stored and exported as a meta data. Valid service names are (see
RFC-67 for `Service Identifier`):

* `kv`
* `query`
* `search`
* `analytics`
* `management`
* `eventing`

The SDK must only account network operations that are visible to server. In
other words it must not increment counter for virtual/compound operations like
`GET_ALL_REPLICAS` and instead track all `GET_REPLICA` sub-operations
individually. During retry/retransmit each operation will be tracked
separately, and transactions will be visible as a set of unrelated subdocument
operations.

The SDK must infer timeout condition on the lower level by pushing timeout
duration from the higher level API, or calculate it from the current time and
the operation deadline, even in case when the SDK core does not have access to
the timer object associated with the request.

## App Telemetry Activation

The collection and reporting of application telemetry should be activated by
either of the following mechanisms:

* presence of the `appTelemetryPath` property of the node in `nodexExt` entry
  of the configuration that contains path on the management port that could be
  upgraded to WebSocket protocol. In this case the SDK would either use
  hard-coded path or extract one from the JSON configuration and use for and
  build the endpoint address the websocket connection.

* presence of the `app_telemetry_endpoint` property in the connection option,
  or similar option in the `ClusterOptions`:

      couchbases://abc.cloud.couchbase.com?app_telemetry_endpoint=ws://localhost:9090/app_telemetry

      ClusterOptions options;
      options.appTelemetryEndpoint("ws://localhost:9090/app_telemetry");

  This endpoint must not have authentication, and should support WebSocket
  protocol.

Below is an exceprt from the server configuration that exposes WebSocket endpoint:

    $ curl -s -u Administrator:password http://192.168.106.128:8091/pools/default/b/default | \
        jq '.nodesExt[0] | pick(.hostname, .services.mgmt, .services.mgmtSSL, .appTelemetryPath)'
    {
      "hostname": "192.168.106.128",
      "services": {
        "mgmt": 8091,
        "mgmtSSL": 18091
      },
      "appTelemetryPath": "/_appTelemetry"
    }

The SDK must not collect metrics if the feature is disabled with
`enable_app_telemetry` or when the cluster does not advertize
`appTelemetryPath` property for at least one node and the user does not specify
`app_telemetry_endpoint`.

## Network Interaction

Once the SDK determined the endpoint it should establish HTTP connection and
upgrade it to WebSocket.

The server allows to close connection at any point of time.

Once the server closes connection, after backoff interval (see section below),
the SDK should pick the next available endpoint, and reconnect.

The SDK must either implement or enable WebSocket PING frame, and respond to
server request with WebSocket PONG whenever the server requests it.

The SDK must respond to any WebSocket PING frame with PONG frame whenever it is
possible to avoid termination of the connection.

The SDK might also send PING frames to the server and expect PONG frame. If the
server does not respond to SDK-initiated PING request, the SDK should close
connection, and reconnect after backoff interval (see section below).

## Reconnection Backoff Interval

To prevent server overload, the SDK must implement an exponential backoff
strategy when reconnecting to the WebSocket.

Upon receiving a configuration update, the SDK must shuffle the list of
endpoints.

The SDK should not apply backoff during the initial iteration of the endpoints.
However, once all endpoints have been tried, it must use an exponential backoff
calculation for subsequent attempts.

The backoff interval must start at `100ms` and gradually increase toward
`app_telemetry_backoff` with each unsuccessful reconnection attempt.

## Wire Protocol

The structure of the protocol is intentionally simple, and yet allows for
future extension.

The SDK leverages WebSocket framing, so does not need to have special fields
for the payload size.

First byte is reserved for Command codes in requests from the server, and for
Status for SDK responses.

This RFC defines the following types of the commands.

| Name            | Value  | Description                                                                  |
|-----------------|--------|------------------------------------------------------------------------------|
| `GET_TELEMETRY` | `0x00` | Retrieve Application Telemetry from SDK in Prometheus Text Exposition Format |

And the following statuses:

| Name              | Value  | Description                                                |
|-------------------|--------|------------------------------------------------------------|
| `SUCCESS`         | `0x00` | No errors has been occurred                                 |
| `UNKNOWN_COMMAND` | `0x01` | The SDK does not support operation requested by the server |


### `GET_TELEMETRY`

The request should not have payload.

The response might have payload if the SDK has metrics to report.

The SDK must reset local counters after successful scheduling response for the
network delivery.

The format of the response follows the [Text-based exposition format of the
Prometheus](https://prometheus.io/docs/instrumenting/exposition_formats/#text-based-format),
and includes metrics and histograms collected by the SDK since the previous
report.

#### Metrics

    sdk_kv_r_timedout{agent="couchbase-net-sdk/2.4.5.0 (clr/4.0.30319.42000) (os/Microsoft Windows NT 10.0.16299.0)",node="node1",node_uuid="91442eb8202e0e16bbb59624d9ccdb0a",bucket="travel-sample"} 1

Where

* `sdk_kv_r_timedout` is a name of the metric, one of the:

  | Metric | Description |
  | ---- | ---- |
  | <code style="white-space:nowrap">sdk_{service}_r_timedout</code> | Timeout |
  | <code style="white-space:nowrap">sdk_{service}_r_canceled</code> | Canceled |
  | <code style="white-space:nowrap">sdk_{service}_r_total</code> | Total number of operations. Note that this number might differ from `_count` of the histogram, if the response from the server has never been received. |

  The `{service}` is one of the following:
  * `kv`
  * `query`
  * `search`
  * `analytics`
  * `management`
  * `eventing`

* `{agent="couchbase-net-sdk/2.4.5.0 (clr/4.0.30319.42000) (os/Microsoft Windows NT 10.0.16299.0)",node="node1",node_uuid="91442eb8202e0e16bbb59624d9ccdb0a",bucket="travel-sample"}` is a list of the tags. The server is free to strip/transform any tags before relaying to Prometheus collector.

  | Label | Description |
  | ---- | ---- |
  | `agent` | agent string of the SDK, as it is used everywhere (HELLO message, logs etc.) |
  | `node` | The hostname (without port) of the node as seen in `nodesExt[].hostname` of the configuration. |
  | `alt_node` | The hostname (without port)  of the node as seen in `nodesExt[]["alternateAddresses"]["<network>"]["hostname"]`. (if present) |
  | `node_uuid` | The UUID of the node as it is seen in `nodesExt[].nodeUUID` of the configuration. |
  | `bucket` | Name of the bucket (if present) |

  It is possible for a `node` or `node_uuid` label to be empty (or absent) if a
  client has yet to receive a config prior to an operation timing out or being
  canceled. The SDK should still try to collect as many metrics as possible,
  and leave it to collector to filter and reject data with empty/stale node
  UUIDs.

* `1` is a metric value. Each metric value increment represents a single
  network operation (e.g. `GET_ALL_REPLICAS` should be counted as `1 +
  number_of_replicas`). The value must be integer (Note that it is more
  restrictive than Prometheus specification, which allows floats for metric
  values, but using integers makes it easier for the server to implement
  aggregation).

Note: while Prometheus allows to specify timestamp in the sample, the SDK do
not need to include it as the Server will erase it anyway, and is not going to
use it as the metrics will be aggregated anyway. The EBNF of the format from
the Prometheus reference.

    metric_name [
      "{" label_name "=" `"` label_value `"` { "," label_name "=" `"` label_value `"` } [ "," ] "}"
    ] value [ timestamp ]

#### Histograms

    sdk_kv_retrieval_duration_milliseconds_bucket{le="1",agent="sdk/2.4.5.0",bucket="travel-sample",node="node1",node_uuid="91442eb8202e0e16bbb59624d9ccdb0a"} 24054
    sdk_kv_retrieval_duration_milliseconds_bucket{le="10",agent="sdk/2.4.5.0",bucket="travel-sample",node="node1",node_uuid="91442eb8202e0e16bbb59624d9ccdb0a"} 33444
    sdk_kv_retrieval_duration_milliseconds_bucket{le="100",agent="sdk/2.4.5.0",bucket="travel-sample",node="node1",node_uuid="91442eb8202e0e16bbb59624d9ccdb0a"} 100392
    sdk_kv_retrieval_duration_milliseconds_bucket{le="500",agent="sdk/2.4.5.0",bucket="travel-sample",node="node1",node_uuid="91442eb8202e0e16bbb59624d9ccdb0a"} 129389
    sdk_kv_retrieval_duration_milliseconds_bucket{le="1000",agent="sdk/2.4.5.0",bucket="travel-sample",node="node1",node_uuid="91442eb8202e0e16bbb59624d9ccdb0a"} 133988
    sdk_kv_retrieval_duration_milliseconds_bucket{le="2500",agent="sdk/2.4.5.0",bucket="travel-sample",node="node1",node_uuid="91442eb8202e0e16bbb59624d9ccdb0a"} 139823
    sdk_kv_retrieval_duration_milliseconds_bucket{le="+Inf",agent="sdk/2.4.5.0",bucket="travel-sample",node="node1",node_uuid="91442eb8202e0e16bbb59624d9ccdb0a"} 144320
    sdk_kv_retrieval_duration_milliseconds_sum{agent="sdk/2.4.5.0",bucket="travel-sample",node="node1",node_uuid="91442eb8202e0e16bbb59624d9ccdb0a"} 53423000
    sdk_kv_retrieval_duration_milliseconds_count{agent="sdk/2.4.5.0",bucket="travel-sample",node="node1",node_uuid="91442eb8202e0e16bbb59624d9ccdb0a"} 144320

Where

* `24054` is a number of the points in the bucket
* `sdk_kv_retrieval_duration_milliseconds_bucket` is a name of the histogram, where
  `_bucket` is a special suffix, that signifies that the line should be
  interpreted as a bucket of the histogram, and
  `sdk_{service}_retrieval_duration_milliseconds` prefix describes particular metric
  of the service. There are two special metrics associated with each histogram:
  `_sum` and `_count` that represent the overall sum of all buckets, and total
  number of the points in the histogram. The `_sum` value should be a total sum
  of the points in milliseconds represented as an integer.
* `le="1"` label is the upper bound of the bucket. All SDK must use hard
  coded buckets to allow easy aggregation. The value would fall into all
  buckets where upper bound is higher. The following buckets should be used
  by the SDKs (note that bucket label have to be treated as a string literal,
  i.e. no trailing zeroes):

The SDK should use `sdk_kv_mutation_durable` for the operation that use
mutation operations that described in RFC-46 "Synchronous Replication". Any
other mutation should use `sdk_kv_mutation_nondurable`. All other operations
should use `sdk_kv_retrieval`.

The SDK should consult [`protocol/mcbp/opcode.cc`](https://github.com/couchbase/kv_engine/blob/master/protocol/mcbp/opcode.cc) of `kv_engine`,
all operations that have `Attribute::ClientWritingData` or `Attribute::Durability`
associated must be considered as mutations.

<table>
  <thead>
    <tr>
      <th>Name</th>
      <th colspan="7">Buckets</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>sdk_kv_retrieval</td>
      <td>&lt;=1ms<br><code style="white-space:nowrap">le="1"</code></td>
      <td>&lt;=10ms<br><code style="white-space:nowrap">le="10"</code></td>
      <td>&lt;=100ms<br><code style="white-space:nowrap">le="100"</code></td>
      <td>&lt;=500ms<br><code style="white-space:nowrap">le="500"</code></td>
      <td>&lt;=1s<br><code style="white-space:nowrap">le="1000"</code></td>
      <td>&lt;=2.5s<br><code style="white-space:nowrap">le="2500"</code></td>
      <td>&gt;2.5s<br><code style="white-space:nowrap">le="+Inf"</code></td>
    </tr>
    <tr>
      <td>sdk_kv_mutation_nondurable</td>
      <td>&lt;=1ms<br><code style="white-space:nowrap">le="1"</code></td>
      <td>&lt;=10ms<br><code style="white-space:nowrap">le="10"</code></td>
      <td>&lt;=100ms<br><code style="white-space:nowrap">le="100"</code></td>
      <td>&lt;=500ms<br><code style="white-space:nowrap">le="500"</code></td>
      <td>&lt;=1s<br><code style="white-space:nowrap">le="1000"</code></td>
      <td>&lt;=2.5s<br><code style="white-space:nowrap">le="2500"</code></td>
      <td>&gt;2.5s<br><code style="white-space:nowrap">le="+Inf"</code></td>
    </tr>
    <tr>
      <td>sdk_kv_mutation_durable</td>
      <td>&lt;=10ms<br><code style="white-space:nowrap">le="10"</code></td>
      <td>&lt;=100ms<br><code style="white-space:nowrap">le="100"</code></td>
      <td>&lt;=500ms<br><code style="white-space:nowrap">le="500"</code></td>
      <td>&lt;=1s<br><code style="white-space:nowrap">le="1000"</code></td>
      <td>&lt;=2s<br><code style="white-space:nowrap">le="2000"</code></td>
      <td>&lt;=10s<br><code style="white-space:nowrap">le="10000"</code></td>
      <td>&gt;10s<br><code style="white-space:nowrap">le="+Inf"</code></td>
    </tr>
    <tr>
      <td>sdk_query</td>
      <td></td>
      <td>&lt;=100ms<br><code style="white-space:nowrap">le="100"</code></td>
      <td>&lt;=1s<br><code style="white-space:nowrap">le="1000"</code></td>
      <td>&lt;=10s<br><code style="white-space:nowrap">le="10000"</code></td>
      <td>&lt;=30s<br><code style="white-space:nowrap">le="30000"</code></td>
      <td>&lt;=75s<br><code style="white-space:nowrap">le="75000"</code></td>
      <td>&gt;75s<br><code style="white-space:nowrap">le="+Inf"</code></td>
    </tr>
    <tr>
      <td>sdk_search</td>
      <td></td>
      <td>&lt;=100ms<br><code style="white-space:nowrap">le="100"</code></td>
      <td>&lt;=1s<br><code style="white-space:nowrap">le="1000"</code></td>
      <td>&lt;=10s<br><code style="white-space:nowrap">le="10000"</code></td>
      <td>&lt;=30s<br><code style="white-space:nowrap">le="30000"</code></td>
      <td>&lt;=75s<br><code style="white-space:nowrap">le="75000"</code></td>
      <td>&gt;75s<br><code style="white-space:nowrap">le="+Inf"</code></td>
    </tr>
    <tr>
      <td>sdk_analytics</td>
      <td></td>
      <td>&lt;=100ms<br><code style="white-space:nowrap">le="100"</code></td>
      <td>&lt;=1s<br><code style="white-space:nowrap">le="1000"</code></td>
      <td>&lt;=10s<br><code style="white-space:nowrap">le="10000"</code></td>
      <td>&lt;=30s<br><code style="white-space:nowrap">le="30000"</code></td>
      <td>&lt;=75s<br><code style="white-space:nowrap">le="75000"</code></td>
      <td>&gt;75s<br><code style="white-space:nowrap">le="+Inf"</code></td>
    </tr>
    <tr>
      <td>sdk_eventing</td>
      <td></td>
      <td>&lt;=100ms<br><code style="white-space:nowrap">le="100"</code></td>
      <td>&lt;=1s<br><code style="white-space:nowrap">le="1000"</code></td>
      <td>&lt;=10s<br><code style="white-space:nowrap">le="10000"</code></td>
      <td>&lt;=30s<br><code style="white-space:nowrap">le="30000"</code></td>
      <td>&lt;=75s<br><code style="white-space:nowrap">le="75000"</code></td>
      <td>&gt;75s<br><code style="white-space:nowrap">le="+Inf"</code></td>
    </tr>
    <tr>
      <td>sdk_management</td>
      <td></td>
      <td>&lt;=100ms<br><code style="white-space:nowrap">le="100"</code></td>
      <td>&lt;=1s<br><code style="white-space:nowrap">le="1000"</code></td>
      <td>&lt;=10s<br><code style="white-space:nowrap">le="10000"</code></td>
      <td>&lt;=30s<br><code style="white-space:nowrap">le="30000"</code></td>
      <td>&lt;=75s<br><code style="white-space:nowrap">le="75000"</code></td>
      <td>&gt;75s<br><code style="white-space:nowrap">le="+Inf"</code></td>
    </tr>
  </tbody>
</table>


#### Examples

The following diagram shows `GET_TELEMETRY (0x00)` request from the server to SDK:

```
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
Transmission Control Protocol, Src Port: 8091, Dst Port: 52370, Seq: 209, Ack: 177535, Len: 3
WebSocket
    1... .... = Fin: True
    .000 .... = Reserved: 0x0
    .... 0010 = Opcode: Binary (2)
    0... .... = Mask: False
    .000 0001 = Payload length: 1
    Payload
Data (1 byte)
    Data: 00
    [Length: 1]

0000  00 00 00 00 00 00 00 00 00 00 00 00 08 00 45 00   ..............E.
0010  00 37 20 59 40 00 40 06 1c 66 7f 00 00 01 7f 00   .7 Y@.@..f......
0020  00 01 1f 9b cc 92 1d 1b b6 e1 52 f6 c7 0d 80 18   ..........R.....
0030  02 e5 fe 2b 00 00 01 01 08 0a 3d 00 3d 05 3d 00   ...+......=.=.=.
0040  39 1c 82 01 00                                    9....
```

And below the response with status `SUCCESS (0x00)` from SDK to the server (the
payload is truncated, and included as unmasked):

```
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
Transmission Control Protocol, Src Port: 52370, Dst Port: 8091, Seq: 224959, Ack: 212, Len: 41169
WebSocket
    1... .... = Fin: True
    .000 .... = Reserved: 0x0
    .... 0010 = Opcode: Binary (2)
    1... .... = Mask: True
    .111 1111 = Payload length: 127 Extended Payload Length (64 bits)
    Extended Payload length (64 bits): 88579
    Masking-Key: d176c493
    Masked payload
    Payload
Data (88579 bytes)
    [Length: 88579]

Unmasked data (88579 bytes):
00000  00 73 64 6b 5f 6b 76 5f 72 5f 74 6f 74 61 6c 7b   .sdk_kv_r_total{
00010  61 67 65 6e 74 3d 22 72 75 62 79 2f 31 2e 32 2e   agent="ruby/1.2.
00020  33 20 28 52 75 62 79 2f 33 2e 33 2e 35 29 22 2c   3 (Ruby/3.3.5)",
00030  62 75 63 6b 65 74 3d 22 64 65 66 61 75 6c 74 22   bucket="default"
00040  2c 68 6f 73 74 3d 22 65 78 61 6d 70 6c 65 2e 63   ,host="example.c
00050  6f 6d 22 7d 20 31 34 33 39 32 20 31 37 33 31 33   om"} 14392 17313
00060  31 34 30 35 37 39 35 39 0a 73 64 6b 5f 71 75 65   14057959.sdk_que
00070  72 79 5f 72 5f 74 6f 74 61 6c 7b 61 67 65 6e 74   ry_r_total{agent
00080  3d 22 72 75 62 79 2f 31 2e 32 2e 33 20 28 52 75   ="ruby/1.2.3 (Ru
00090  62 79 2f 33 2e 33 2e 35 29 22 2c 62 75 63 6b 65   by/3.3.5)",bucke
000a0  74 3d 22 64 65 66 61 75 6c 74 22 2c 68 6f 73 74   t="default",host
000b0  3d 22 65 78 61 6d 70 6c 65 2e 63 6f 6d 22 7d 20   ="example.com"}
...
```


## Integration with Custom Meters

When the user might overrides the default meter with their own (e.g. providing
OpenTelemetry meter), the reporting features, described in this RFC must not be
disabled and the SDK must still collect and report metrics regarding
application and library health.

## Configuration

This SDK does not require any additional configuration properties with
Committed status. Although for the faster adoption and easier testing, it
should implement the following options and document them as Volatile.

* `enable_app_telemetry` Option to control whether the feature is enabled or
not. When the feature is disabled, the SDK will ignore telemetry endpoints in
the configuration. The default value should be `true`.

* `app_telemetry_endpoint` that allows to override the reporting endpoint
  discovered through configuration, or provide one if the configuration does
  not announce support for the Application Telemetry.

* `app_telemetry_backoff` the maximum time duration to wait before WebSocket
  reconnection. Once exponential backoff calculator will approach this maximum,
  it must stop increasing the backoff interval and remain constant. The default
  value should be `1 hour`.

* `app_telemetry_ping_interval` the time duration to wait between consecutive
PING commands to the server. The default value should be `30 seconds`. The
default scraping interval on the server is 60 seconds, and we want to avoid the
server is using dead WebSocket as the server cannot initiate connection back to
the SDK. As a solution the ping interval from the SDK should be smaller than
the scraping interval.

* `app_telemetry_ping_timeout` the time to allow the server to respond to PING
WebSocket command before destroying the connection. The default value should be
`2 seconds`.

# References


# Signoff

| Language   | Team Member            | Signoff Date   | Revision |
|------------|------------------------|----------------|----------|
| .NET       | Emilien Bevierre       | 2025-04-08     | 2        |
| C++        | Sergey Avseyev         | 2025-04-07     | 2        |
| Go         | Dimitris Christodoulou | 2025-04-07     | 2        |
| Java       | David Nault            |                |          |
| Kotlin     | David Nault            |                |          |
| Node.js    | Jared Casey            | 2025-04-07     | 2        |
| PHP        | Sergey Avseyev         | 2025-04-07     | 2        |
| Python     | Jared Casey            | 2025-04-07     | 2        |
| Ruby       | Sergey Avseyev         | 2025-04-07     | 2        |
| Scala      | Graham Pople           | 2025-04-08     | 2        |

# Changelog

* November 11, 2024 - Revision #1 (by Sergey Avseyev)
    * Initial Draft

* April 7, 2024 - Revision #2 (by Sergey Avseyev)
    * Clarify reconnection backoff behavior.
    * Update `app_telemetry_backoff` default to `1 hour`

[text-exposition-format]: https://prometheus.io/docs/instrumenting/exposition_formats/
