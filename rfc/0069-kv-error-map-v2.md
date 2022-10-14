# Meta

- RFC Name: KV Error Map V2
- RFC ID: 0069-kv-error-map-v2
- Start Date: 2021-09-17
- Owner: Brett Lawson
- Current Status: DRAFT
- Supersedes
  - SDK-RFC#13 (KV Error Map)

# Summary

This proposal outlines the SDK handling of error categories and attributes defined by Couchbase Server 5.0 and above (through [kv-engine](https://github.com/couchbase/kv_engine)), including the changes introduced by Couchbase Server 7.0 for Error Map V2.

# Motivation

Prior to the introduction of Error Map, error codes were explicitly defined, or occasionally were defined as ranges of values. In many cases, new server versions would introduce new error codes which older clients were unaware of. This would lead to clients responding to error situations in unexpected ways, and in some cases leading to clients which were unable to make forwards progress. Consequently, error code changes were difficult to introduce to the server, in spite of the potential to significantly improve behaviour (for example, it is much more helpful to return an error code `LOCKED` if an item is locked, than a `TMPFAIL` which was done previously). Starting in Couchbase Server 5.0, clients are able to request a mapping of the error codes that the server will use when communicating with us, along with various meta-data about those errors and how they should be handled. Starting in Couchbase Server 7.5, this error map has been enhanced with some additional guidelines and rules to ensure its continued usefulness.

# General Design

The server exposes the error map as a simple JSON blob that contains the mappings of known error codes to their attributes. The client requests this error map from the server during the negotiation phase (after `HELLO`). When the server responds with an error code that the client otherwise does not know how to handle, it refers to the error map and determines (semi-heuristically) the best course of action.

## Control Flow

1.  The client must first advertise that it supports the KV Error Map feature by including the `XERROR (0x07)` feature in the HELLO negotiation command. This tells the server that it may send newer error codes to the client without risking confusing it. It also implies that the client will retrieve the error code map from the server.
2.  Once the client has sent the `XERROR` feature and receive indication that the server also supports it (by receiving `XERROR` included in the server's list of features), it sends a `CMD_GET_ERROR_MAP (0xfe)`, indicating the maximum format version of the error map the client understands. There are currently two supported versions: 1 (Couchbase Server 5.0+) and 2 (Couchbase Server 7.5+). Clients should always request the highest version that they support, though the server may respond with a lower version and this lower version must still be parsed and used.
3.  The server will reply with a JSON error map. The client must store this error map on a per node basis. The version and revision are contained in the error map, as well as the actual code-to-attribute mappings. The version is used to indicate new error attributes, whereas the revision is used to indicate updated error codes.
4.  When the client receives an unrecognized error code, it can consult the error map (as below).

## Negotiation

The client should first indicate that it supports the KV Error Map feature by sending the `XERROR (0x07)` feature flag in `HELLO`. If the server responds back with `XERROR` in the list of features, then proceed to the next step. Otherwise, the server doesn't support extended error codes.

_Note:_ `HELLO` should be done as the first step, even before auth, so that auth can return more detailed error information. This is known to break certain old and unsupported server versions, so if possible an escape hatch like a flag should be introduced which performs `HELLO` after authentication to keep them working.

**The XERROR bit should be sent by default.** Since the client is under authority when to consult the error map, more information is always good but the clients need to ensure that backwards compatibility is not broken. So in some cases newer error codes or attributes need to be mapped back to old behavior when encountered to make sure the semantics of the SDK don't change with different server versions.

## Getting the Error Map

To get the error map, send the `CMD_GET_ERROR_MAP (0xfe)` to the server. This should be done immediately after the client has received acknowledgment that the server actually supports extended error codes, as the server is now free to send any error code it wants in reply.

The `GET_ERROR_MAP` command requires a 2 byte payload, which is the network-encoded maximum version number the client can parse. At the time of this RFC, there are two supported versions. Clients should always request the highest version they support. The server should send back a reply with the JSON error map in the payload. Note that the error map payload version returned from the server may not be the same as was requested in the case of an older server version. These older versions must still be parsed and used.

## Parsing the Error Map

The error map's format is defined in the [memcached Github repository](https://github.com/couchbase/kv_engine/blob/master/docs/ErrorMap.md), and will not be repeated here again. The client should parse the error map and ensure that all error attributes are understood and accounted for. If the error map seems corrupted in any way, it should ignore it, and renegotiate with the server (or reconnect) without the `XERROR` feature.

Once the error map has been received, it should be stored in a per-node database. Different nodes may wish to dictate how different errors behave (for example, in respect to different cluster versions and/or retries).

## Handling Error Attributes

Error attributes beyond the retry attributes (discussed further down), which is utilized to indicate that a client must use the retry semantics of the particular error are not in scope for this document. A future RFC will provide details on additional attributes that may be added, and the behaviors they will provide. If in doubt if an attribute should be used, the SDK should use a conservative approach and favor backwards compatibility.

# Error Attributes

Error Categories act as the driving force behind the utility of the error map. Categories define the basic characteristics of an error condition so that even if the SDK does not understand a specific error code, it may determine the next course of action based on the error's defined attributes.

Attributes are the most important part of the error map as they convey how an error code should be handled. An error code may have more than a single attribute.

- `item-only`: This attribute means that the error is related to a constraint failure regarding the item itself, i.e. the item does not exist, already exists, or its current value makes the current operation impossible. Retrying the operation when the item's value or status has changed may succeed.
- `invalid-input`: This attribute means that a user's input was invalid because it violates the semantics of the operation, or exceeds some predefined limit.
- `fetch-config`: The client's cluster map may be outdated and requires updating. The client should obtain a newer configuration.
- `conn-state-invalidated`: The current connection is no longer valid. The client must reconnect to the server. Note that the presence of other attributes may indicate an alternate remedy to fixing the connection without a disconnect, but without special remedial action a disconnect is needed.
- `auth`: The operation failed because the client failed to authenticate or is not authorized to perform this operation. Note that this error in itself does not mean the connection is invalid, unless conn-state-invalidated is also present.
- `special-handling`: This error code must be handled specially. If it is not handled, the connection must be dropped.
- `support`: The operation is not supported, possibly because the of server version, bucket type, or current user.
- `temp`: This error is transient. Note that this does not mean the error is retriable.
- `internal`: This is an internal error in the server.
- `retry-now`: The operation may be retried immediately.
- `retry-later`: The operation may be retried after some time.
- `subdoc`: The error is related to the subdocument subsystem.
- `dcp`: The error is related to the DCP subsystem.
- `rate-limit`: The error is related to rate limiting being applied to the current connection or user.

The most up-to-date list of attributes can to be found under https://github.com/couchbase/memcached/blob/master/docs/ErrorMap.md#error-attributes.

It is important to note that a single error code may contain more than one attribute. Some attributes are informational by nature, while others may affect the flow of the SDK, and yet others may affect the flow of the application. Additionally, to support clean forwards compatibility, the SDKs must be designed such that unrecognized attributes are silently ignored and behaviours defined for any of the known attributes are still applied.

It is important to reiterate while the attributes are important for unknown error codes to decide what to do next, the SDKs must favor backwards compatibility for known status codes / responses, and the error map attributes should only be used to decide in the case that an unknown status code is encountered.

# Retry Intervals

While the error map includes various values which can be used for the specification of client retry behaviour, clients should not currently make use of these values instead opting to use the configured retry strategy in the SDK.

# Processing Chain

Making use of the error map consists of the SDK consistently and intelligently determining the best course of action as well as the best error to report for any error code depending on its specific attributes.

## Processing Order Example

Note that this RFC does not specify exactly how each SDK should handle all the attributes (to follow in a new RFC). Here are some examples that might apply:

1.  _Special-handling._ This error must be specially handled by the client. The connection must be dropped and reinitialized if it can't handle the error properly.
2.  Fetch-config: Hint that the client may wish to get a new cluster map. This attribute does not affect the processing of the given error, but may help avoid receiving such an error in the future.
3.  Temporary: Informational attribute that may be relayed to the user -- the operation may be retried at some point in the future without any modification
4.  Item-only: Informational attribute that may be relayed to the user
5.  Auth: AuthZ or AuthN error (usually the former). Informational attribute that may be relayed to the user.
6.  Conn-state-invalidated: Client must drop the connection and reconnect. This may be ignored if the client otherwise knows how to handle the error (e.g. `AUTH_STALE`).

# Reserved Ranges

While the error map allows the server to technically send us any error within the range of 0-65536, and clients must handled any error code within this range, the range 0xff00-0xffff is reserved for testing/proxy use and will never be included in an error map dispatched by the server. This should have no impact on the SDK implementation.

# Language Specifics

Currently there are no known language specifics for this RFC.

# Questions

**(Unknown) Should HELLO or AUTH be done first?**
We can HELLO at any time. HELLO should be done first, as a result. Note that in many cases, many of our initial bootstrap operations can be pipelined together as a single stream of commands. Note: older servers may not be happy with HELLO first and disconnect us, this is clarified in the RFC above now.

**(Matt) How are mixed version clusters handled? 4.6 and 5.0 nodes? 5.0 and 5.1 nodes?**
The client will keep track of the error map on a per-node basis so that each node has the highest version of the error map it can support. In practice its expected that the changes are additive so its going to be a minor detail (mn).

**(Unknown) Precedence of SDK settings over cluster settings?**
SDK settings must always take precedence and only unknown response status codes should be looked up. The SDK is the authority.

**(Unknown) How does the client request a compatible version?**
At the protocol level, the client can just write the version maximum version it desires as part of the "key".

**(Unknown) How is logging handled for each of these? Predefined? Part of the response? Goal, I think, is common behaviors.**
Logging is determined based on the general SDK logging facilities. Errors are normally somewhere at WARN while others might be DEBUG. The SDK should just hook this up into the usual logging flow.

**(Unknown) What happens if an outer operation is specified by the user to be 300s, but the `MaxDuration` of the retry specification specifies something different?**
The shorter time between the two options should always take precedence. Note that the retry specification can specify a `MaxDuration` of 0 to indicate to follow the client behavior.

**(Brett) Which error map should the client be using when it processes the error map for sub-document operations? Is it safe to use the highest possible version?**
The client should utilize the highest revision error map that it has on a per node basis, in the same manner that it processes error maps for all other kinds of status codes.

**(Brett) What happens if the error code changes between retry behaviors?**
If the error code changes between retry behaviors, than the new retry behavior should be utilized, or the operation failed immediately in the case of a non-retriable error. Note that the retry count should be reset when a new, different error code is received.

**(Brett) What should the strategy be if the client receiving the error map cannot handle that retry possibility or behavior attribute?**
Bubble the error back to the user. We should never end up in this kind of situation though, because the version exchanged in the GET_ERROR_MAP guarantees the server will not give the client a retry strategy that it does not understand. In any case, the unknown attribute should be ignored.

# Changelog

- Sept 17, 2021 - Revision \#1 (by Brett Lawson)

  - Copied 0013-kv-error-map which covers Error Map V1.

- Sept 17, 2021 - Revision \#2 (by Brett Lawson)

  - Updated 'Summary' section to indicate specific coverage of v2 error maps.
  - Updatec 'Control Flow' to cover the new version number that should be used when requesting error maps.
  - Updated 'Getting the Error Map' to cover the new version number.
  - Updated 'Error Attributes' section with the newly introduced `rate-limit` attribute.
  - Updated 'Error Attributes' section to clarify that unrecognized attributes must be ignored by the SDK.
  - Updated 'Error Attributes' section to clarify that attributes should only be used to decide SDK behaviour
    when unknown status codes are received, and SDK-defined behaviour should always be used for known status codes.
