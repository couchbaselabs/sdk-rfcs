# Meta

- RFC Name: SDK3 Diagnostics
- RFC ID: 0061-sdk3-diagnostics
- Start Date: November 12, 2019
- Owner: Michael Nitschinger \<michael.nitschinger@couchbase.com\>
- Current Status: ACCEPTED
- Relates To:
  - SDK-RFC#59 (SDK3 Foundation)
- [Original Google Drive Doc](https://docs.google.com/document/d/1Lw3nuYVtRbXYIujeCxakcyYQWFs2OAj4Borb75eNe5Y)

# Motivation

This SDK3 RFC describes the various ping and diagnostic operations available throughout the SDK.

# Summary

The various diagnostic options are available at multiple API levels throughout the SDK.

```java
enum ClusterState {
  Online, // all nodes and their sockets are reachable
  Degraded, // at least one socket per service is reachable
  Offline, // not even one socket per service is reachable
}

enum EndpointState {
  Disconnected, // the endpoint socket is not reachable
  Connecting, // currently connecting (includes auth,...)
  Connected, // connected and ready
  Disconnecting, // disconnecting (after being connected)
}
```

## Cluster-Level

```java
class Cluster {

  ...

  void waitUntilReady(Duration timeout, [WaitUntilReadyOptions options])

  DiagnosticsResult diagnostics([DiagnosticsOptions options])
  PingResult ping([PingOptions options])
}
```

## Bucket-Level

```java
class Bucket {

  ...

  void waitUntilReady(Duration timeout, [WaitUntilReadyOptions options])

  PingResult ping([PingOptions options])
}
```

## The Diagnostics Command

Creates diagnostic report that can be used to determine the healthfulness of the cluster. It does not proactively perform any I/O against the network.

Signature

```typescript
DiagnosticsResult Diagnostics([DiagnosticsOptions options]);
```

Parameters

- Required
  - None
- Optional
  - reportId - an optional string name for the generated diagnostics report.

Returns

- A DiagnosticsReport object detailing the current status of the SDK.

Throws

- `CouchbaseException` if anything unexpected comes up (like invalid arguments), but since it is doing I/O usually only validation errors will be thrown

#### DiagnosticsResult JSON Output

The SDK Must provide a method on the DiagnosticsResult to export it to JSON into this format:

```json
{
  "version": 2, // This is a uint, and is always 2
  "id": "0xdeadbeef",
  "sdk": "gocbcore/v7.0.2",
  "services": {
    "search": [
      {
        "id": "0x1415F11",
        "last_activity_us": 1182000,
        "remote": "centos7-lx1.home.ingenthron.org:8094",
        "local": "127.0.0.1:54669",
        "state": "reconnecting",
        "details": "RECONNECTING, backoff for 4096ms from Fri Sep  1 00:03:44 PDT 2017"
      }
    ],
    "kv": [
      {
        "id": "0x1415F12",
        "last_activity_us": 1182000,
        "remote": "centos7-lx1.home.ingenthron.org:11210",
        "local": "127.0.0.1:54670",
        "state": "connected",
        "namespace": "bucketname"
      }
    ],
    "query": [
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
    "analytics": [
      {
        "id": "0x1415F15",
        "last_activity_us": 1182000,
        "remote": "centos7-lx1.home.ingenthron.org:8095",
        "local": "127.0.0.1:54675",
        "state": "connected"
      }
    ],
    "views": [
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

## The waitUntilReady Command

Waits until a desired cluster state by default ("online") is reached or times out.

This command lives both at the Cluster and at the Bucket level. At the cluster level it waits until cluster-level resources are ready (query, search,... and gcccp connections if available). At the bucket level it ALSO includes bucket-level resources (view connections and bucket-scoped kv connections).

Signature

```java
void waitUntilReady(Duration timeout, [WaitUntilReadyOptions options]);
```

Parameters

- Required
  - Timeout: The timeout duration the user is willing to wait
- Optional
  - desiredState - ClusterState the desired state (default ONLINE).
  - serviceTypes - the set of service types to check, if not provided will use all services from the config

Throws

- `CouchbaseException` if anything unexpected comes up (like invalid arguments)
- `UnambiguousTimeoutException` if the expected state cannot be reached in the given timeout

`waitUntilReady` will perform a ping if it has not seen activity on a specific service. If there has been activity it will use this information to determine readiness (i.e. by using diagnostics). See the `ClusterState` enum that defines the semantics when to return for the desired state.

Optionally, the SDK may ensure only a single call through synchronization or a lock, and/or return the same object to await on multiple calls.

Note: It is a hard error if OFFLINE is passed in, but keeping it for future use (i.e. returning a cluster state somewhere). In addition, it is also present at any async or reactive APIs with their equivalent Async<T> return types.

For now, no checks are added to the Collection level.

## The ping Command

Actively performs I/O by application-level pinging services and returning their pinged status. This API is available on the Cluster and on the Bucket level.

Signature

```java
PingResult ping([PingOptions options]);
```

Parameters

- Required
  - None
- Optional
  - reportId - an optional string name for the generated ping report
  - timeout - the timeout for all ping commands (if no timeout is chosen, the individual timeouts will be used as configured on the config)
  - serviceTypes - the set of service types to check, if not provided will use all services from the config

Returns
A PingResult object representing the result of each ping performed.

Throws

- `CouchbaseException` if anything unexpected comes up (like invalid arguments) (note that timeouts must not be propagated but will be surfaced as individual errors in the report)

#### PingResult

```java
interface PingResult {
  String Id();
  Int16 Version(); // VERSION IS 2
  String Sdk();
  Map<ServiceType, List<EndpointPingReport>> Endpoints();
}

interface EndpointPingReport {
  ServiceType Type();
  String Id();
  String Local();
  String Remote();
  PingState State();
  Optional<String> Error(); // if ping state is error, contains the error message what appened
  Optional<String> Namespace();
  Duration Latency();
}

enum PingState {
  Ok,
  Timeout,
  Error
}
```

#### PingResult JSON Output

The SDK Must provide a method on the PingResult to export it to JSON into this format:

```json
{
  "version": 2,
  "id": "0xdeadbeef",
  "config_rev": 53,
  "sdk": "gocbcore/v7.0.2",
  "services": {
    "sSearch": [
      {
        "id": "0x1415F11",
        "latency_us": 877909,
        "remote": "centos7-lx1.home.ingenthron.org:8094",
        "local": "127.0.0.1:54669",
        "state": "ok"
      }
    ],
    "kKv": [
      {
        "id": "0x1415F12",
        "latency_us": 1182000,
        "remote": "centos7-lx1.home.ingenthron.org:11210",
        "local": "127.0.0.1:54670",
        "state": "ok",
        "namespace": "bucketname"
      }
    ],
    "query": [
      {
        "id": "0x1415F14",
        "latency_us": 2213,
        "remote": "centos7-lx2.home.ingenthron.org:8095",
        "local": "127.0.0.1:54682",
        "state": "timeout"
      }
    ],
    "analytics": [
      {
        "id": "0x1415F15",
        "latency_us": 2213,
        "remote": "centos7-lx1.home.ingenthron.org:8095",
        "local": "127.0.0.1:54675",
        "state": "error",
        "error": "endpoint returned HTTP code 500!"
      }
    ],
    "views": [
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

## Enumerations

### ServiceType (type: any)

- KV (json: "kv")
- Query (json: "query")
- Analytics (json: "analytics")
- Search (json: "search")
- Views (json: "views")
- Manager (json: "mgmt")

# Changelog

- Nov 12, 2019 - Revision #1 (by Michael Nitschinger)

  - Initial Draft

- Nov 18, 2019 - Revision #2 (by Michael Nitschinger)

  - Added waitUntilReady

- Dec 12, 2019 - Revision #3 (by Michael Nitschinger)

  - Make timeout mandatory on waitUntilReady

- Dec 16, 2019 - Revision #4 (by Michael Nitschinger)

  - Renamed Services() to Endpoints() for consistency because it is called EndpointDiagnostics
  - Changed the Key of the endpoints() map from String to ServiceType - since we define this enum already it is much easier for the user to get it that way rather than a very generic String type. Also since we can have many endpoints per service type, the API has been changed to a list. Result:\
    Before Map<String, EndpointDiagnostics>\
    After Map<ServiceType, List<EndpointDiagnostics>>
  - Added EndpointState enum similarly to ClusterState so it can be observed at the individual level too (and we need to turn it into a string anyways for the JSON export)
  - Moved "scope" to "namespace" because scope is now an overloaded term with collections
  - Changed lastActivity simple int64 to Duration to align with the other RFCs on how to represent a time duration. Also changed to an optional, since on the diagnostics API there might have not been activity on that endpoint yet.

- Dec 17, 2019 - Revision #5 (by Michael Nitschinger)

  - Removed ping as its own command and is subsumed by the ping option on the diagnostics command
  - Added a separate command section for waitUntilReady

- Dec 18, 2019 - Revision #6 (by Michael Nitschinger)

  - Removed ping option again from diagnostics
  - Re-added and clarified on the ping command
  - Reorganized the rfc and cleanup

- Dec 21 2019 - Revision #7 (by Michael Nitschinger)

  - Added options to ping
  - Moved ping result back to a List since it can still return a list of EndpointPingReports if you have multiple nodes

- April 16, 2020 - Revision #8 (by Michael Nitschinger)

  - Removed version from DiagnosticResult, made it clear in the JSON output that value type is of uint and always 2. Note: if you've already implemented it, you can deprecate it and comment here.
  - Added ServiceType enumeration

- April 30, 2020 (by Michael Nitschinger)

  - Moved RFC to ACCEPTED state.

- Sept 17, 2021 (by Brett Lawson)

  - Converted to Markdown

# Signoff

| Language   | Team Member         | Signoff Date | Revision |
| ---------- | ------------------- | ------------ | -------- |
| Node.js    | Brett Lawson        | 2020-04-16   | 8        |
| Go         | Charles Dixon       | 2020-04-22   | 8        |
| Connectors | David Nault         | 2020-04-29   | 8        |
| PHP        | Sergey Avseyev      | 2020-04-22   | 8        |
| Python     | Ellis Breen         | 2020-04-29   | 8        |
| Scala      | Graham Pople        | 2020-04-30   | 8        |
| .NET       | Jeffry Morris       | 2020-04-23   | 8        |
| Java       | Michael Nitschinger | 2020-04-16   | 8        |
| C          | Sergey Avseyev      | 2020-04-22   | 8        |
| Ruby       | Sergey Avseyev      | 2020-04-22   | 8        |
