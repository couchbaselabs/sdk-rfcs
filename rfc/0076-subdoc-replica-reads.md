# Meta

| Field          | Value                      |
|----------------|----------------------------|
| RFC Name       | Sub-Document Replica Reads |
| RFC ID         | 76                         |
| Start Date     | 2023-06-20                 |
| Owner          | Charles Dixon              |
| Current Status | DRAFT                      |
| Revision       | #1                         |

# Summary
[MB-23162](https://issues.couchbase.com/browse/MB-23162) adds support for performing subdocument read operations on replica servers.
Replica read operations are performed against replica servers with a new ReplicaRead protocol flag.
it also includes a new, informational, HELLO flag to be negotiated.

This RFC specifies the SDK work needed to support this feature.
All SDKs are required to implement this feature.

# Motivation
SDKs require a way to expose this new ReplicaRead feature in a way that is idiomatic to the SDK API.

# Changes

1. Add two new functions to the `Collection` interface - `LookupInAnyReplica` and `LookupInAllReplicas`.
2. Add a new `LookupInReplicaResult` result type.
3. Add two new options types - `LookupInAnyReplicaOptions` and `LookupInAllReplicasOptions`.
4. Support the new REPLICA_READ (0x1c) HELLO flag.
5. When the above functions are used send `LookupIn` requests to replica nodes specifying the REPLICA_READ protocol flag (0x20).

## LookupInAllReplicas

Gets a stream of document data from the server using LookupIn, leveraging both the active and all available replicas; the caller can choose which replica they wish to take and cancel the stream at any time.
This MUST fetch from both the active and all replicas due to failover situations that can arise.
In addition, the SDK MUST broadcast the requests simultaneously to ensure the application receives data as quickly as possible.

```
Stream<ILookupInReplicaResult> LookupInAllReplicas(string id, LookupInSpec[] specs, [LookupInAllReplicasOptions])
```

Parameters

- Required

    - Id: string - the primary key.
    - LookupInSpec - an array of fetch operations, see [LookupIn](./0053-sdk3-crud.md#LookupIn)

- Optional
    - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
    - Serializer - a custom serializer for converting the memcached JSON data to a native type.
    - ParentSpan - the parent span for any top level spans created during this operation.
    - RetryStrategy - the strategy to apply for retries.

Returns

A stream of replicas as a ILookupInReplicaResult object

Throws

- Documented

    - RequestTimeoutException (#1)
    - CouchbaseException (#0)
    - FeatureNotAvailableException (#15)

- Undocumented
    - InvalidArgumentException (#3)

## LookupInAnyReplica

Gets a document for a given id, leveraging both the active and all available replicas.
This method follows the same semantics of LookupInAllReplicas (including the fetch from ACTIVE), but returns the first response as opposed to returning all responses.

Signature

```
ILookupInReplicaResult LookupInAnyReplica(string id, LookupInSpec[] specs, [LookupInAnyReplicaOptions])
```

Parameters

- Required

    - Id: string - the primary key.
    - LookupInSpec - an array of fetch operations, see [LookupIn](./0053-sdk3-crud.md#LookupIn)

- Optional

    - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
    - Serializer - a custom serializer for converting the memcached JSON data to a native type.
    - ParentSpan - the parent span for any top level spans created during this operation.
    - RetryStrategy - the strategy to apply for retries.

Returns

- A ILookupInReplicaResult wrapping a ILookupInResult object.

Throws

NOTE: The following exceptions are thrown at the "top" level. The errors that are thrown on a per-field basis are covered in [LookupIn](./0053-sdk3-crud.md#LookupIn).

- Throws
    - Documented
        - DocumentUnretrievableException (#125)
            - The situation where we got responses from all of them (most likely: key not found) but none of them were SUCCESS so we ended up not returning anything (empty stream from lookupin all replicas).
        - RequestTimeoutException (#1)
        - CouchbaseException (#0)
        - FeatureNotAvailableException (#15)
    - Undocumented
        - InvalidArgumentException (#3)

## ILookupInReplicaResult

The ILookupInReplicaResult interface is the return type for Sub-Document read operations from replicas.

```csharp
public interface ILookupInReplicaResult : ILookupInResult {
  T ContentAs<T>(int lookupIndex);
  Boolean Exists(int lookupIndex);
  Boolean IsReplica();
}
```

See [LookupIn](./0053-sdk3-crud.md#LookupIn) for information on `Exists` and `ContentAs`.

- IsReplica
  - Returns true if the read was from a replica node.

## BucketCapabilities Check
The `bucketCapabilities` field within the bucket config object must contain `subdoc.ReplicaRead` in order to use the `ReadReplica` document flag.

It will only contain this field when ns_server has determined that all nodes have been upgraded to 7.5 or above.

## HELLO
SDKs must implement and send a new HELLO flag with the code 0x1c.
This HELLO flag is informative and does not actually enable/disable the feature, there is no reason to expose a way to disable this flag.

Only Couchbase Server 7.5 and above will negotiate it. 
However, it can be negotiated by a 7.5 memcached node, when the cluster overall is not fully upgraded to 7.5.

Hence, the reliable capability check is the “bucketCapabilities” one.
At the point this is returned, all memcached nodes should be upgraded to 7.5 or above.
This renders the HELLO check largely unnecessary (the HELLO doesn’t activate anything in memcached either, it is purely informational) but can be useful in debugging.

## Sending the request
At the point just before sending the request, if the “bucketCapabilities” check fails (see above), then the SDK should raise a FeatureNotAvailableException.

The check is done on each request, to support clusters being online-upgraded to 7.5.

To perform ReplicaReads the SDK sends a normal LookupIn request to the relevant replica node and sets the top-level document flag to enable flag 0x20.
Of course, this should be OR-ed with other such flags.

## Observability updates

### New top level span/metric names

| Operation type      | Identifier             |
|---------------------|------------------------|
| LookupInAnyReplica  | lookup_in_any_replica  |
| LookupInAllReplicas | lookup_in_all_replicas |

Each of these operations must also include N inner `lookup_in` spans per replica/active read.

# Changelog
* June 21 2023 - Revision #1
  * Initial Draft

# Signoff

| Language    | Team Member    | Signoff Date | Revision |
|-------------|----------------|--------------|----------|
| .NET        | Jeffry Morris  | 2023-07-25   | #1       |
| C/C++       | Sergey Avseyev | 2023-06-27   | #1       |
| Go          | Charles Dixon  |  2023-05-21            |    #1      |
| Java/Kotlin | David Nault    | 2023-07-06   | #1       |
| Node.js     | Jared Casey    | 2023-07-19           | #1        |
| PHP         | Sergey Avseyev | 2023-06-27   | #1       |
| Python      | Jared Casey    | 2023-07-19              | #1          |
| Ruby        | Sergey Avseyev | 2023-06-27   | #1       |
| Scala       | Graham Pople   | 2023-07-04 | #1 |

# Reference
[KV patchset](https://review.couchbase.org/c/kv_engine/+/186113)
