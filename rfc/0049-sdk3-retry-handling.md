# Meta

- RFC Name: SDK3 Retry Handling
- RFC ID: 0049-sdk3-retry-handling
- Start Date: Aug 19, 2019
- Owner: Michael Nitschinger \<michael.nitschinger@couchbase.com\>
- Current Status: ACCEPTED
- [Original Google Drive Doc](https://docs.google.com/document/d/1h9YL2k8uzNPnxN_XanagEc2d8qRVu6sVDiKaqASqjS4)

# Motivation

Every application eventually has to deal with failed operations, and many of these concerns are the same across SDK.s This RFC aims to clarify when failures happen and how to retry them in a controlled manner.

> "If you fail to plan, you are planning to fail!" - Benjamin Franklin

# Introduction

Distributed systems are complex, and failures can occur at any point in time. As a result, the user cannot expect every operation to succeed all the time and needs to plan for failure.

From the SDKs perspective, failure can occur at certain discrete stages in the lifetime of an operation:

1. **Client Dispatch:** Before being sent over the network (example: no socket available)
2. **Network Time:** After sending it, but before retrieving a response (example: socket being close while operation is in-flight)
3. **Response Received:** After receiving a response (example: a non-success response status from the server)

```
|--- Client Dispatch --->
                          |--- Network Time --->
                                                 |--- Response Received --->
|---------------------------- Request Lifetime ---------------------------->
```

While all of them have the same outcome (the operation did not succeed), the impact on a potential retry effort is very different. Independent of the actual failure type, the important distinction is if a retry attempt is changing the state of the wider system in a deterministic fashion.

Note that determinism is a bit different than idempotency (an operation is idempotent that has no additional effect if it is called more than once with the same input parameters). Non-deterministic operations are not always safe to retry:

- Client Dispatch: Yes (no global state has been mutated on the server yet)
- Network Time: No (without a response, we do not know if any state changed or not)
- Response Received: It depends (dependant on the type of operation and the response code)

Idempotent operations are always safe to retry. As an example, a KV get operation would never mutate server state, so it can be retried safely in all 3 of those stages. Compared to a KV upsert operation, which can only be retried as long as it is not sent over the network. As soon as it is on the wire, the client has no control if the upsert has been applied or not, risking data loss if retried blindly (in case someone else modified the same document in between).

# (No) Reordering Guarantees

A potential reordering of operation becomes more likely in the face of failures, so it is important to clarify the guarantees (and non-guarantees) the SDK can provide.

At the time of writing, the SDKs do not make any guarantees w.r.t total ordering of operations. Also, the effect of an operation following an other can only be as strong as the guarantees of the Server. So even if operation X is completed successfully, it needs to use the correct durability requirements so that when operation Y is dispatched it would see change X or later (but never X-1).

Even when no failures are identified in the system, reordering can happen: if operations need to be retried because of KV Not-My-Vbucket (or similar), it can happen that an operation will complete later than the one scheduled behind originally.

Finally, because locking across multiple user threads is expensive, the SDKs do not try to enforce reordering guarantees for individual documents/operations globally in the application.

**As a result, the SDK does not provide any reordering guarantees for pipelined operations.** If a user needs to rely on the result of one operation for the next one, they need to wait until the first operation succeeds. Any parallelism/pipelining will not be reordered semantically.

When talking about determinism in this RFC, we specifically mean knowing if and what cause an operation caused on the Couchbase Server cluster (i.e. a document has been modified, or deleted).

# Best Effort Retry vs. Fail Fast

Independent of the stage where an operation failed, there needs to be a decision made between one of the two cases:

- Retry an operation in a "best effort" way until it succeeds or something else prevents it from being retried (i.e. a user-configured timeout)
- Failing fast immediately and not retrying.

Both strategies have their use, and some applications might even need to change on a per operation basis. As a result, a default strategy is advised that will suit most of the users in the stock configuration but it must be overridable globally and on a per-operation basis (see implementation guidelines for more information).

# Different Stage Retry Semantics

The following section discusses retry semantics for each of those stages.

## Client Dispatch

This is the initial phase of an operation, before it is sent over the network. In this stage both idempotent and non-idempotent can be retried without non deterministic behavior.

Common causes of retry needed in the client dispatch state are:

- Network failures (all sockets down for a given node)
- Service not available (i.e. no query node present after failover)
- Sockets not writable (i.e. too many outstanding writes)

In addition, there might be other states the SDK is in that are implementation specific that can make an operation non dispatchable.

## Network Time

As soon as an operation is sent on the network, there is no way for the SDK to know if a non-idempotent operation has modified the cluster state or not. As a result, only idempotent operations can be retried at this stage.

**All non-idempotent operations must be failed to avoid data loss.**

## Response Received

If a response is received and it does not return with a status code indicating "success", the non-success codes fall into the following categories for non-idempotent operations:

- Not-Retryable (i.e. permanent errors like "invalid durability level", generic failure from an unknown response code, etc.)
- Retryable with domain context (errors which indicate that the operation has not modified any server state, i.e. temporary failure, locked,...)
- Retryable with application context (i.e. a replace failing with doc not found might be retried if the user knows it will exist on retry)

Note that just because an operation can be retried, doesn't mean the user wants it to or it is advisable for the cluster health in the first place. Temporary failures, locked documents and similar might make sense to retry in some applications while it doesn't in others (they rather want to propagate the error or switch to an alternate datasource quickly).

Then there are responses which the SDK doesn't even want to surface to the user and retry under any circumstances: i.e. "not my vbucket". These need to be retried even if the user specifies a "fail fast" policy since it is considered transient and a transparent retry will very likely turn into a subsequent successful response (but might still not if there is a bug in the system, so it needs to be accounted for).

# Implementation Details

This section covers the common implementation guidelines across the SDKs.

## Request Annotation

- each request has a "idempotent" property/method accessible. This allows both the user and automated policies to decide if it is safe to retry a certain operation or not in the first place, depending on when they have been sent into retry handling.
- each request also has an associated retry strategy, which is either the default one or a per-request overridden one by the user (see the next section for more information).
- The retryAttempts method returns the number of retries that already happened, which allows retry strategies to calculate backoff delays)

```java
interface Request {
  bool idempotent()
  RetryStrategy retryStrategy()
  int retryAttempts()
  Set<RetryReason> retryReasons(); // <- previous retry reasons for this request
  void incrementRetryAttempts();
}
```

This information is then used by the RetryOrchestrator and the strategy to decide if - and how - an operation is being retried.

## Retry Strategies

The RetryStrategy at its core is an interface which allows the RetryOrchestrator to decide on if and how an operation should be retried. It can be implemented by the user, but is considered an advanced API. The SDK must provide one implementation out of the box:

- A "best effort" retry strategy

The best effort retry strategy is the default one and will retry as many operations (if possible safely) as possible until their respective timeout hits (or because of some other SDK-specific reasons is not eligible for being retried anymore). Since most of the errors are considered transient, this is going to give the user the best experience out of the box (since many retry cases under failure scenarios are already covered for them).

```java
interface RetryStrategy {
  Future<RetryAction> retryAfter(Request request, RetryReason reason);
}
```

The interface is simple: with the request and the reason for retry in scope, it returns a retry action which contains the duration  when the next retry attempt should happen. If none (or null) duration is returned, then no retry is allowed and the request needs to be failed.

Note that it returns a Future<RetryAction> instead of a plain RetryAction because we want to give the user a chance in custom implementations to perform IO to external systems when making a decision. If it isn't  an async interface, the user would unintentionally end up blocking in our internal constructs. If the SDK can guarantee that this does not happen, it may also just return the RetryAction directly.

```java
interface RetryAction {
  Optional<Duration> duration();
}
```

By having the duration wrapped into an action, it allows for future extensibility where (also SDK-specific) certain actions might be defined that the SDK understands in addition to the plain retry.

### Accessible Context in the RetryStrategy

To make the RetryStrategy more useful when the user wants to override it, the retryAfter method needs to provide additional context on the request. This context can either be accessible through the Request itself (first argument) or it needs to be passed in as a third optional argument that is platform specific.

One example of this context is, depending on the platform, to let the user access user-defined data on the request. In java, there is a RequestContext associated with every Request which holds a Map that the user can fill with data. They could apply logic as follows if needed:

```java
Map<String, Object> myData = request.context().clientContext();
if (myData.contains("isRobotRequest")) {
  return Future.complete(RetryAction.noRetry());
} else {
  return super.retryAfter(request, reason);
}
```

So if somewhere in their web stack they decided (maybe based on a header) that the request comes from a robot they do not want to retry the operation at all and make it consume additional time and resources. Otherwise it falls back to the best effort retry strategy.

### Best Effort

The best effort retry strategy aims to be the sensible default for most users and a stock configuration. It tries to complete an operation until it succeeds or it times out.

It should use an exponential delay that retries the first attempts fairly quickly but then takes a bit longer to not overwhelm the system. Boundaries between 1ms and 500ms. Note that while this is the default retry strategy implementation, there should be a constructor available to pass in a different backoff calculator so that a user can customize the retry delays to their needs.

```java
Future<RetryAction> retryAfter(Request request, RetryReason reason) {
  if (request.idempotent() || reason.allowsNonIdempotentRetry() {
    return Future.complete(RetryAction.withDuration(\
      Optional.of(calculateBackoff(request.retryAttempts())
    ));
  }
  return Future.complete(RetryAction.noRetry()); // not eligible for retry
}
```

A reference backoff calculator implementation can be found in the java source code [here](https://github.com/couchbase/couchbase-jvm-clients/blob/colossus-sr-4/core-io/src/main/java/com/couchbase/client/core/retry/reactor/Backoff.java).


### Fail Fast (Strict)
For the first several years of SDK3, a fast fail strategy was not provided out-of-the-box.

While the implementation of a fail fast strategy is rather simple, we made a conscious decision to NOT include it by default.

(The rest of this section is left verbatim for historical context, but note that a targeted fast-fail strategy that fails only on the most useful errors is now defined below.)

The reasoning goes as follows:

1. The BestEffort strategy, being the default, will serve most of our users needs out of the box
2. Modifying the RetryStrategy is an advanced use-case and not part of the beginner experience. Once you start tinkering with it, it's very likely you are at least somewhat familiar with the SDK already
3. If we would provide it out of the box and users use it, they'll see much more exception come up and wanting to handle them manually in try/catch blocks, promoting copy/pasting of those blox and not leading to DRY code in the end
4. Rather, they should extend the BestEffort retry strategy and only override the specific situations they need to.

If users really need this strategy, they can implement it trivially like this on their own:

```java
Future<RetryAction> retryAfter(Request request, RetryReason reason) {
  return Future.complete(RetryAction.noRetry());
}
```

Finally, SDKs itself might include an internal version of the FailFast retry strategy to use it for their commands where it makes sense (i.e. cccp config polling). If such a strategy is present, it must be marked as Internal

### Fail Fast (Terminal Errors)
A `FastFailOnTerminalErrorRetryStrategy` is provided by the SDKs as an opt-in alternative to `BestEffortRetryStrategy`, and intended to provide a significantly improved developer experience.

It was created after multiple years of end-user experience with SDK3, and as a result for most users it is likely the better option.  Any new SDKs should default to it, and our documentation should reflect this as a standard part of SDK usage.

There are four goals:

* To be a better experience for most users, by providing immediate clear feedback on basic problems such as bad credentials or resources not existing, rather than timeouts.
* To fast-fail only on "terminal errors", as very few users truly want a strategy that fast-fails on literally everything including NOT_MY_VBUCKET errors.  Although the notion of terminality is hard to pin down since it depends greatly on application context, we can informally define it as "errors where a fast-failure will be more useful than a timeout to the majority of users, and where an immediate-term retry is unlikely to make any progress".
* To permit improving `FastFailOnTerminalErrorRetryStrategy` over time, so that it always fulfils its remit of providing the best possible user experience including when new server functionality is added.  To this end it will intentionally be marked as `@Stability.Volatile` indefinitely, and it will be documented that users wishing to hardcode specific retry and error behaviours should copy the class.
* To provide clearer and more user-facing RetryReasons.

To achieve these goals, the following RetryReasons are added.  The first two apply to all operations, KV or HTTP:

* AUTHENTICATION_ERROR: if any current topology-tracking connection (*) has previously received a 0x2 error at the memcached SASL auth step.  This per-connection state gets cleared if authentication subsequently succeeds for that connection.  Any generic SASL problem that is not an explicit 0x2 server error does not result in AUTHENTICATION_ERROR, so it is reserved for an explicit indication that the user's credentials are incorrect.
* TLS_ERROR: if any current topology-tracking connection has previously received any form of explicitly TLS-related failure from the underlying SSL library.  The most common scenario will be that the server's certificate is not trusted.  In the unlikely scenario that a connection subsequently passes the TLS handshake, then this per-connection state is cleared.

(*) a 'topology tracking connection' being one that is used for cluster config push/poll purposes, whether over memcached (GCCCP) or ns-server.

These three apply only to KV operations, and are not used for any HTTP services, which return their own error messages and codes:

* BUCKET_ACCESS_ERROR: for a given bucket, if any connection has previously received a NO_ACCESS at the memcached SELECT BUCKET step for that bucket.  This global state gets cleared if SELECT BUCKET succeeds for that bucket, which could happen if the bucket subsequently gets created or RBACs added.  Note that the server returns NO_ACCESS both for the bucket not existing, and for the user not having relevant RBACs.
** todo: SELECT BUCKET can also return NOT_FOUND, but the situations in which it does so are currently unclear.
* SCOPE_NOT_FOUND: used if at point of requiring collection map (e.g. to encode document key) for a given KV operation, the map does not have the required scope, AND a map has previously been successfully fetched.
* COLLECTION_NOT_FOUND: used if at point of requiring collection map (e.g. to encode document key) for a given KV operation, the map does not have the required collection, AND a map has previously been successfully fetched.

And `FastFailOnTerminalErrorRetryStrategy` will handle these by mapping:

| RetryReason              | Behaviour |
| -------------------------| -------------------- |
| AUTHENTICATION_ERROR     | Fast fails with AuthenticationFailureException |
| TLS_ERROR                | Fast fails with CouchbaseException |
| BUCKET_ACCESS_ERROR      | Fast fails with CouchbaseException |
| SCOPE_NOT_FOUND          | Fast fails with ScopeNotFoundException |
| COLLECTION_NOT_FOUND     | Fast fails with CollectionNotFoundException |
| Any other                | The same behaviour as BestEffortRetryStrategy |

While it should not matter, for completeness of specification the mapping should be applied in that order e.g. `if AUTHENTICATION_ERROR .. else if TLS ERROR .. else if BUCKET_ACCESS_ERROR..`.

Open questions left to resolve during pathfinding:

* Handling mTLS failures.
* Handling JWT failures.
* Checking handling & failures align with couchbase2.

## RetryReason

The RetryReason is an enumeration which lists possible reasons why an operation needs to be potentially retried in the first place. The reason also has additional information on if the request is in a stage where it might be retryable for non-idempotent operations as well.

```java
enum RetryReason {
  REASON_1(true, true),
  REASON_2(false, false);
  boolean allowsNonIdempotentRetry();
  boolean alwaysRetry();
}
```

| RetryReason                         | Non-idempotent Retry | Always retry |
| ----------------------------------- | -------------------- | ------------ |
| UNKNOWN                             | false                | false        |
| SOCKET_NOT_AVAILABLE                | true                 | false        |
| SERVICE_NOT_AVAILABLE               | true                 | false        |
| NODE_NOT_AVAILABLE                  | true                 | false        |
| KV_NOT_MY_VBUCKET                   | true                 | true         |
| KV_COLLECTION_OUTDATED              | true                 | true         |
| KV_ERROR_MAP_RETRY_INDICATED        | true                 | false        |
| KV_LOCKED                           | true                 | false        |
| KV_TEMPORARY_FAILURE                | true                 | false        |
| KV_SYNC_WRITE_IN_PROGRESS           | true                 | false        |
| KV_SYNC_WRITE_RE_COMMIT_IN_PROGRESS | true                 | false        |
| SERVICE_RESPONSE_CODE_INDICATED     | true                 | false        |
| SOCKET_CLOSED_WHILE_IN_FLIGHT       | false                | false        |
| CIRCUIT_BREAKER_OPEN                | true                 | false        |
| QUERY_PREPARED_STATEMENT_FAILURE    | true                 | false        |
| QUERY_INDEX_NOT_FOUND               | true                 | false        |
| ANALYTICS_TEMPORARY_FAILURE         | true                 | false        |
| SEARCH_TOO_MANY_REQUESTS            | true                 | false        |
| VIEWS_TEMPORARY_FAILURE             | true                 | false        |
| VIEWS_NO_ACTIVE_PARTITION           | true                 | true         |
| AUTHENTICATION_ERROR                | todo                 | todo         |
| TLS_ERROR                           | todo                 | todo         |
| BUCKET_ACCESS_ERROR                 | todo                 | todo         |
| SCOPE_NOT_FOUND                     | todo                 | todo         |
| COLLECTION_NOT_FOUND                | todo                 | todo         |

- UNKNOWN: All unexpected/unknown retry errors must not be retried to avoid accidental data loss and non-deterministic behavior.
- SOCKET_NOT_AVAILABLE: The socket is not available into which the operation should've been written.
- SERVICE_NOT_AVAILABLE: The service on a node (i.e. kv, query) is not available.
- NODE_NOT_AVAILABLE: The node where the operation is supposed to be dispatched to is not available.
- KV_NOT_MY_VBUCKET: A not my vbucket response has been received.
- KV_COLLECTION_OUTDATED: A KV response has been received which signals an outdated collection.
- KV_ERROR_MAP_RETRY_INDICATED: An unknown response was returned and the consulted KV error map indicated a retry.
- SOCKET_CLOSED_WHILE_IN_FLIGHT: While an operation was in-flight, the underlying socket has been closed.
- CIRCUIT_BREAKER_OPEN: The circuit breaker is open for the given socket/endpoint and as a result the operation is not sent into it.

Reasons outlined in this list are considered a guideline and must not be followed exactly (especially if SDK-idiomatic names should be used for clarity) - but SDKs should aim for similarity where possible. Of course, additional reasons might be added depending on implementation specifics.

## Retry Orchestration

The retry orchestrator is the last missing piece in the retry puzzle which handles the actual retry dispatching. It is called from various places in the SDK and then dispatches into the retryStrategy.

```java
class RetryOrchestrator {
  void maybeRetry(Request request, final RetryReason reason);
}
```

The following is a rough guideline of how the orchestrator can work. Of course it needs to be adapted to the specific SDKs implementation details (i.e. the mechanics on how to actually dispatch the retry after delay or how to actually fail a request):

```java
void maybeRetry(Request request, final RetryReason reason) {
  if (reason.alwaysRetry()) {
    dispatchSdkSpecificRetry(controlledBackoff(), request);
    request.incrementRetryAttempts();
    logRetryAttempt(request, reason);
    return;
  }

  RetryAction retryAction = request.retryStrategy().retryAfter(
    request,
    reason
  );
  Optional<Duration> duration = retryAction.duration();
  if (duration.isPresent()) {
    request.incrementRetryAttempts();
    logRetryAttempt(request, reason);
    Duration cappedDuration potentiallyCapDuration(duration);
    dispatchSdkSpecificRetry(cappedDuration, request);
  } else {
    logNotRetried(request, reason);
    failRequest(request);
  }
}
```

Note the explicit requirements to log failed and succeeded retry attempts including the reason so it can be debugged after the fact.

### Controlled Backoff for Always Retry

In order to avoid spamming the cluster in pathological "always retry" cases, a controlled backoff duration must be used that uses the following simple algorithm:

```java
switch (retryAttempt) {
  case 0:
    return Duration.ofMillis(1);
  case 1:
    return Duration.ofMillis(10);
  case 2:
    return Duration.ofMillis(50);
  case 3:
    return Duration.ofMillis(100);
  case 4:
    return Duration.ofMillis(500);
  default:
    return Duration.ofMillis(1000);
}
```

The outcome of this strategy is that early on these requests are retried quickly, but after 4 attempts there will be a one-second delay to avoid spamming the cluster quickly under pathological conditions.

### Capping the Retry Duration

Note that in the RetryOrchestrator there is a code snippet as follows:

```java
Duration cappedDuration potentiallyCapDuration(duration);
```

This is purely an optimization which performs the following logic:

- If the requested retry delay would ultimately exceed the request timeout, it needs to be capped to that limit.

This makes sure that requests are not hanging around longer in the system than they absolutely need to.

As an example: if a KV operation is 2s into its lifetime and a retry duration asks for a 1s "sleep" and the overall KV timeout is 2.5s, it must be capped down to 500ms instead so it does not exceed the operation lifetime.

When the retry duration triggers in this case, it is a given that the operation will time out at that point in time. For optimization purposes the operation should be immediately cancelled after the duration and not sent back into the regular request/response flow.

## KV Error Map Handling

The SDK consults the KV error map for all KV errors it does not know anything about. While consulting the KV error map, the server might indicate that an error is retryable through the retry spec attached ("retry-now" or "retry-later").

To not interfere with the user-configured retry strategy, the following logic must be used:

- If the KV error map indicates it can be retried, it needs to be forwarded to the Retry orchestrator to decide.
- If the KV error map misses any retry indication, it must be cancelled and not be sent to the Retry orchestrator.
- If forwarded, the Orchestrator (and the attached retry strategy) decides if and how the request should be retried.

Additional information from the retry specification (like ceil, interval, max-duration) must not be used if the user specifies this as part of the retry strategy (which implicitly is the case with the default retry strategies shipped).

## Idempotent Operation List

Every operation should be treated as non-idempotent unless they are specified in this list:

- KV

  - Carrier Config & Global Config Requests
  - Get Request (NOT get and lock and NOT get and touch)
  - Get Collection ID
  - Get Collection Manifest
  - Noop Request
  - Observe Via CAS, Observe via Sequence Number
  - Replica Get Request
  - Subdoc Lookup

- Query

  - If the "readonly" property is set to "true" on the Query

- Analytics

  - If the "readonly" property is set to "true" on the Analytics Query

- Search

  - Since FTS is read-only, all search requests are idempotent (even though the actual FTS request is a POST)

- View

  - Since views are read-only, all view requests are idempotent

- Cluster Manager

  - Every "GET" call (read only) is idempotent (i.e. config requests)

- Management APIs
  - Every "GET" call (read only) is idempotent

## Response Code Retry List

By default, once a complete, known response is received it should not be retried, unless specifically outlined in this list.

- KV

  - Not My VBucket 0x07 on all messages (Reason: KV_NOT_MY_VBUCKET)
  - Collection Outdated 0x88 on all messages (Reason: KV_COLLECTION_OUTDATED)
    - **Note:** Explicitly excluded here is the GetCollectionID KV command, since it returns 0x88 if the collection is not found.
  - Locked 0x09 (Reason: KV_LOCKED)
    - **Note:** do NOT retry on KV_LOCKED for the "Unlock" command (the retry handler should never see it in the first place!( Reason is that this needs to be surfaced as a cas mismatch exception to the user and not being retried
  - Temporary Failure 0x86 (Reason: KV_TEMPORARY_FAILURE)
  - SyncWriteInProgress 0xa2 (Reason: KV_SYNC_WRITE_IN_PROGRESS)
  - SyncWriteReCommitInProgress 0xa4 (Reason: KV_SYNC_RE_COMMIT_IN_PROGRESS)
  - **Note:** DCP commands are excluded from this list

- Query

  - Error code 4040 (Reason: QUERY_PREPARED_STATEMENT_FAILURE)
  - Error code 4050 (Reason: QUERY_PREPARED_STATEMENT_FAILURE)
  - Error code 4070 (Reason: QUERY_PREPARED_STATEMENT_FAILURE)
  - Error code 5000 if the message also contains "queryport.indexNotFound" (Reason: QUERY_INDEX_NOT_FOUND)

- Analytics

  - Error code 23000 (Reason: ANALYTICS_TEMPORARY_FAILURE)
  - Error code 23003 (Reason: ANALYTICS_TEMPORARY_FAILURE)
  - Error code 23007 (Reason: ANALYTICS_TEMPORARY_FAILURE)

- Search

  - Http status code 429 (Reason: SEARCH_TOO_MANY_REQUESTS)

- View
  - Http Status code 302 (VIEWS_NO_ACTIVE_PARTITION)
  - Others as VIEWS_TEMPORARY_FAILURE
    - Http status codes: 300, 301, 302, 303, 307, 401,408,409,412,416,417,501,502,503,504
    - Http 404 **IF** the text contains ""reason":"missing""
    - Http 500 **IF NOT**
      - Contains "error" and "{not_found, missing_named_view}"
      - Or Contains "error" and ""badarg""

# Changelog

- Aug 19, 2019 - Revision #1 (by Michael Nitchinger)

  - Added RetryAction and section on KV Error Map Handling

- Aug 20, 2019 - Revision #2 (by Michael Nitchinger)

  - First fill of the retry reason table, Added section on idempotent operation list

- Aug 26, 2019 - Revision #3 (by Michael Nitchinger)

  - Added controlled backoff for the always retry case

- Aug 27, 2019 - Revision #4 (by Michael Nitchinger)

  - Added initial response code retry list based on the 2.x SDK

- Sept 2, 2019 - Revision #5 (by Michael Nitchinger)

  - Added explicit retry reasons for KV response status codes

- Sept 3, 2019 - Revision #6 (by Michael Nitchinger)

  - Added exclusion for GetCollectionID command on the retry response code

- Sept 6, 2019 - Revision #7 (by Michael Nitchinger)

  - Added readonly idempotency option for analytics

- Oct 18, 2019 - Revision #8 (by Michael Nitchinger)

  - Added RetryReason if the circuit breaker has the circuit open

- Nov 8, 2019 - Revision #9 (by Michael Nitchinger)

  - Changed RetryAction to Future<RetryAction> on RetryStrategy
  - Make it explicit that NO fail fast strategy is shipped out of the box
  - Added a section on capping the retry duration
  - Add section on accessible context on the RetryStrategy
  - Updated Response code retry list and gave the HTTP errors explicit names

- Nov 11, 2019 - Revision #10 (by Michael Nitchinger)

  - Added a note that internal fail fast strategies are okay but must be marked as Internal (since internal SDK components might rely on it)

- Nov 15, 2019 - Revision #11 (by Michael Nitchinger)

  - Added a note that KV_LOCKED must not be retried on unlock

- Nov 18, 2019 (by Michael Nitchinger)

  - Moved RFC to REVIEW state.

- Dec 23, 2019 - Revision #12 (by Michael Nitchinger)

  - Added section on VIEWS retry error codes, ported from SDK 2 (reason: VIEWS_TEMPORARY_FAILURE) as well as VIEWS_NO_ACTIVE_PARTITION
  - Added QUERY_INDEX_NOT_FOUND to the table

- April 16, 2020 - Revision #13 (by Michael Nitchinger)

  - Added a note clarifying on the best effort retry backoff calculator (with link to example implementation in java)

- April 30, 2020 - Revision #14 (by Michael Nitchinger)

  - Moved RFC to ACCEPTED state.

- Sept 17, 2021 - Revision #15 (by Brett Lawson)

  - Converted to Markdown

- Nov, 2025 - Revision #16 (by Graham Pople)

  - Added `FastFailOnTerminalErrorRetryStrategy`.

# Signoff

| Language   | Team Member         | Signoff Date | Revision |
| ---------- | ------------------- | ------------ | -------- |
| Node.js    | Brett Lawson        | 2020-04-16   | 13       |
| Go         | Charles Dixon       | 2020-04-22   | 13       |
| Connectors | David Nault         | 2020-04-29   | 13       |
| PHP        | Sergey Avseyev      | 2020-04-22   | 13       |
| Python     | Ellis Breen         | 2020-04-29   | 13       |
| Scala      | Graham Pople        | 2020-04-30   | 13       |
| .NET       | Jeffry Morris       | 2020-04-22   | 13       |
| Java       | Michael Nitschinger | 2020-04-16   | 13       |
| C          | Sergey Avseyev      | 2020-04-22   | 13       |
| Ruby       | Sergey Avseyev      | 2020-04-22   | 13       |
