# Meta
- RFC Name: Health Check
- RFC ID: 0034
- Start Date: 2017-09-06
- Owner: Sergey Avseyev
- Current Status: ACCEPTED

# Summary
There are situations where an end user does not control the entire network and
they want a way to check the health of connectivity. Also sometimes an end user
needs to keep connections alive in environments where they'll be shut down. This
change describes new API calls and internal changes required to provide an SDK
health report and optionally force idle connections to remain alive.

# Use Case Analysis
There are many different ways an API like this can be used, but they can be
distilled broadly into two categories:

* Proactive Pull-Based monitoring systems with health check information on the
  spot:
    * Kubernetes Liveness and Readiness Probes via http or cli command
      (https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)
    * Docker Health Check with cli commands
      (https://docs.docker.com/engine/reference/builder/#healthcheck)
    * AWS ELB through http
      (http://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-healthchecks.html)
* Reactive log and log aggregation components for later analysis:
    * Logging the state into logfiles on regular intervals for later manual
      analysis.
    * Feeding this state either from logs or directly into monitoring systems on
      intervals like splunk, elasticsearch or nagios.

Based on these use cases it makes sense to expose two kinds of APIs which both
represent the same underlying state. A raw JSON payload which can be fed into
the reactive log and aggregation components, as well as a strongly typed API in
each language which handles the first scenario and for example allows to distill
a "200 OK" response from the complex service state of an SDK.

# Description
The following sections describe the individual fields which are part of the JSON
payload as well as the `DiagnosticsReport` part of the actual API. Later
sections introduce both the raw JSON as well as the API.

## Version
- Mandatory, Integer

The report layout version. Currently it must always be `1`.

## ID
- Mandatory, String

The report must have `"id"` property, which might be configurable by SDK users,
and automatically generated otherwise (It should be at least unique in scope of
the application, but UUID is also fit). This field is able to be specified by
the user in the function call.

## Configuration revision
- Mandatory for `ping()`, Not used for `diagnostics()`, Integer

The `ping()` report should contain Reportrevision of the configuration that the
SDK is actively using when the report is generated. It also helps to determine
the full list of nodes.

## SDK identifier
- Mandatory, String

The same as identifier seen in HELO command and User-Agent in HTTP requests.

## Services
- Mandatory, Array

On the top level we have `"services"` key, which contains map of service keys, to
arrays of the endpoints.

| Service Key | Description                                                                                                                              |
|-------------|------------------------------------------------------------------------------------------------------------------------------------------|
| `"kv"`      | Memcached binary protocol sockets                                                                                                        |
| `"config"`  | If configuration colling mechanism uses dedicated sockets, they should use separate key (even though they technically `"kv"` or `"http"` |
| `"mgmt"`    | Sockets for management requests (administering the cluster or bucket properties                                                          |
| `"view"`    | Couchbase View queries                                                                                                                   |
| `"n1ql"`    | N1QL queries                                                                                                                             |
| `"fts"`     | Full text search (CBFT) queries                                                                                                          |
| `"cbas"`    | Analytics queries                                                                                                                        |

### Sevice Id
- Mandatory, String

The service object must have some string identifier (`"id"`), which is unique in
scope of resource container (the entity which generates report). For example, it
could be a socket descriptor, or corresponding in-memory address of wrapping
structure.

The Service id is a logical identifier for the same "socket structure", and
should stay the same across reconnect attempts for example (even if the
`"local"` field would change on a reconnect attempt).

When SDK does not control the socket directly, but still willing to expose
connection metrics, it could use the connection hashcode or object id here, so
that the user will be able to track connection throughout the several consequent
reports (in scope of report ID).

### Service state
- Mandatory, String

The list of service states provided here defines an exhaustive list, and it is
expected that not all SDKs are able to show all of these states, BUT they should
not define new states that are not part of this list. If additional information
is required, it can be placed in the "details" field.

The `"state"` field describes the current connection conditions. Here are all the
possible fields for the Diagnostics API:

* `"new"`: the connection is initialized but it has never been connected (or
  trying to) yet
* `"connecting"`: identifies the connection in the initial connect attempt
* `"authenticating"`: state during connecting or reconnecting where the client
  is actively performing the authentication (i.e. SASL).
* `"connected"`, normal case, everything operating
* `"disconnected"`, always indicated unexpectedly
* `"reconnecting"`: trying to reconnect after a first initial connect attempt
* `"disconnecting"`, planned, if say the map changed or the cluster requested a
  graceful connection shut down; some requests may still be in flight.

In case of PING API, the states defined as following:

* `"timeout"`: the server wasn't able to reply to ping request in time. The time
  should be the same as for corresponding service request.
* `"ok"`: the service responded with successful code, see `"latency_us"` for
  latency
* `"error"`: some error happened (more information might be supplied in
  `"details"`)

### Scope
- Mandatory for scoped services, String

On services which are scoped to a bucket, then the `"scope"` field needs to
reflect the bucket name. On services where there is no bucket scoping, the field
can be omitted (in practice, right now the scope field is needed for kv services
where the value is the bucket name).

### Service Latency
- Mandatory for `ping()` report, Integer

This field must specify latency time in microseconds. It has to be specified
when active keep-alive used (`ping()` API). Ignored for `diagnostics()` API.

### Service Last Activity
- Mandatory for `diagnostics()` report, Integer

The field must specify the time in microseconds since relative last activity of
the service (not absolute as in epoch). This field is mandatory, if activity
happened yet and can be omitted if none has happened yet (since 0 would be
ambiguous as there might have been traffic at the time of the health
check). Optional for ping report.

### Service Local Address
- Optional, String

This field must contain endpoint local address in `"host:port"` form. This field
is optional as not all HTTP libraries expose socket API.

### Service Remote Address
- Mandatory, String

This field must contain remote address that matches what was supplied in the
configuration from the cluster and not be modified in the SDK (like resolve it),
which `$HOST` substitution if necessary. This field is mandatory.

### Service Details
- Optional, String

The endpoint entry of the JSON might contain optional field `"details"` with the
string, describing current state of the endpoint. It might be a humanized
description of the current state (status code for example, next time to
reconnect etc.). Those details can also contain SDK specific information and are
not standardized as part of this RFC.

## Library keep-alive mechanism (Ping)

The service ping request should be implemented as separate function. It should
not send request to every socket when the SDK using pools. The goal is to make
sure, that services are reachable. It must use corresponding timeout interval
for the type of the service.

The SDK must ping every service from the configuration (all KV nodes, all N1QL
nodes etc.).

Ping implementation for various services (they should be as close as possible to
real behaviour, i.e. use same sockets/pool):

* Memcached service - `NOOP`
* N1QL - `GET /admin/ping` (expect 200 OK)
* CBFT - `GET /api/ping` (expect 200 OK)
* Views - `GET /` (expect 200 OK)
* Analytics - `GET /admin/ping` (will return 404 for <=DP4 releases, and 200 OK for others)
* Config and Management - not applicable

If the configuration does not have service defined, the SDK will not send ping
requests, and will not include node into report. If for some service, there no
nodes to ping at all, then the report will not include corresponding section for
the service.

See service state definition for possible values of `"state"` property for ping
responses.

## API

The `diagnostics()` API must be defined on the level, which own resources. For
example, it does not make much sense to define API on bucket level, when SDK has
shared pool of HTTP connections on the cluster level, because all of them have
to be included in the report. The SDK might choose not to include service
information into report, when it is not possible to gather information without
active checks.

Note that users of the `diagnostics()` API may determine health from the
diagnostics report returned. While some users may want a "health check", the
name of this function and the return value were chosen to be clear that it does
not attempt to give a boolean healthy true/false to the consumer.  The
diagnostic report is a rough, backwards view that an application developer can
use to determine healthy for their specific workload.  SDKs do not have enough
context to determine a boolean healthy/unhealthy, so the goal with this API is
to summarize as much info as possible for the app developer to assemble a
complete, contextual view and come to a conclusion.

The `ping()` function must be defined on the bucket level, as it generates new
requests. Ping requests are not initiated by SDK, the user should call API
explicitly. It uses timeouts configured for the services.

There should be a function, which will return an object, which could be
serialized to JSON string, or return the JSON string explicitly.

This is mandatory part of API, and along with JSON format, should be implemented
by all SDKs.

    interface DiagnosticsReport {
      // export into the later defined JSON standard format
      // if a language has idioms for json export, go use that.
      String exportToJson();
    }

    interface PingReport {
      // export into the later defined JSON standard format
      // if a language has idioms for json export, go use that.
      String exportToJson();
    }

    // ask all services
    DiagnosticsReport diagnostics(String reportId);

    // use autogenerated report ID
    DiagnosticsReport diagnostics();

    // ping all services when services list is empty
    PingReport ping(String reportId, ServiceType... services);

    PingReport ping(ServiceType... services);

For service type, the SDK might use already defined enums which represents the
possible services, or use the following:

    enum ServiceType {
      View,
      KeyValue,
      Query,
      Config,
      Search,
      Analytics
    }

It is allowed and recommended to implement more API on the `DiagnosticsReport`
and `PingReport` classes for convenience, but they must be marked as
experimental.

## The sample `diagnostics` JSON

``` json
{
  "version": 1,
  "id": "0xdeadbeef userstring",
  "sdk": "gocbcore/v7.0.2",
  "services": {
    "fts": [
      {
        "id": "0x1415F11",
        "last_activity_us": 1182000,
        "remote": "centos7-lx1.home.ingenthron.org:8094",
        "local": "127.0.0.1:54669",
        "state": "reconnecting",
        "details": "RECONNECTING, backoff for 4096ms from Fri Sep  1 00:03:44 PDT 2017"
      }
    ],
    "kv": [
      {
        "id": "0x1415F12",
        "last_activity_us": 1182000,
        "remote": "centos7-lx1.home.ingenthron.org:11210",
        "local": "127.0.0.1:54670",
        "state": "connected",
        "scope" : "bucketname",
      }
    ],
    "n1ql": [
      {
        "id": "0x1415F13",
        "last_activity_us": 1182000,
        "remote": "centos7-lx1.home.ingenthron.org:8093",
        "local": "127.0.0.1:54671",
        "state": "connected"
      },
      {
        "id": "0x1415F14",
        "last_activity_us": 1182000,
        "remote": "centos7-lx2.home.ingenthron.org:8095",
        "local": "127.0.0.1:54682",
        "state": "disconnected"
      }
    ],
    "cbas": [
      {
        "id": "0x1415F15",
        "last_activity_us": 1182000,
        "remote": "centos7-lx1.home.ingenthron.org:8095",
        "local": "127.0.0.1:54675",
        "state": "connected"
      }
    ],
    "view": [
      {
        "id": "0x1415F16",
        "last_activity_us": 1182000,
        "remote": "centos7-lx1.home.ingenthron.org:8092",
        "local": "127.0.0.1:54672",
        "state": "connected"
      }
    ]
  }
}
```

## The sample `ping` JSON

``` json
{
  "version": 1,
  "id": "0xdeadbeef userstring",
  "config_rev": 53,
  "sdk": "gocbcore/v7.0.2",
  "services": {
    "fts": [
      {
        "id": "0x1415F11",
        "latency_us": 877909,
        "remote": "centos7-lx1.home.ingenthron.org:8094",
        "local": "127.0.0.1:54669",
        "state": "ok"
      }
    ],
    "kv": [
      {
        "id": "0x1415F12",
        "latency_us": 1182000,
        "remote": "centos7-lx1.home.ingenthron.org:11210",
        "local": "127.0.0.1:54670",
        "state": "ok",
        "scope" : "bucketname",
      }
    ],
    "n1ql": [
      {
        "id": "0x1415F13",
        "latency_us": 6217,
        "remote": "centos7-lx1.home.ingenthron.org:8093",
        "local": "127.0.0.1:54671",
        "state": "ok"
      },
      {
        "id": "0x1415F14",
        "latency_us": 2213,
        "remote": "centos7-lx2.home.ingenthron.org:8095",
        "local": "127.0.0.1:54682",
        "state": "timeout"
      }
    ],
    "cbas": [
      {
        "id": "0x1415F15",
        "latency_us": 2213,
        "remote": "centos7-lx1.home.ingenthron.org:8095",
        "local": "127.0.0.1:54675",
        "state": "error",
        "details": "endpoint returned HTTP code 500!"
      }
    ],
    "view": [
      {
        "id": "0x1415F16",
        "latency_us": 45585,
        "remote": "centos7-lx1.home.ingenthron.org:8092",
        "local": "127.0.0.1:54672",
        "state": "ok"
      }
    ]
  }
}
```

# Signoff

| Language | Representative      | Date       |
|----------|---------------------|------------|
| C        | Sergey Avseyev      | 2017-12-18 |
| Go       | Brett Lawson        | 2017-12-20 |
| Java     | Michael Nitschinger | 2017-12-18 |
| .NET     | Jeff Morris         | 2017-12-20 |
| Node.js  | Brett Lawson        | 2017-12-20 |
| PHP      | Sergey Avseyev      | 2017-12-18 |
| Python   | Ellis Breen         | 2017-12-18 |
