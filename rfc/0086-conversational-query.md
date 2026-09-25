# Meta

* RFC Name: Conversational Query
* RFC ID: 86
* Start Date: 2026-08-31
* Owner: Anirudh Lakhotia, Jared Casey
* Current Status: DRAFT
* Revision: 4
* Supporting Material: [sdk-design](https://github.com/couchbaselabs/sdk-design/tree/main/server-aligned/totoro/query-begin-chat)

# Summary

Conversational Query allows applications to submit natural-language prompts to the Query service and receive generated SQL++ statements and Query results. This RFC defines one-shot queries, multi-turn chats, model credentials, Knowledge management, and the SDK behavior required to continue or persist a chat across application requests.

# Motivation

Conversational Query supports one-shot natural-language queries and chats that may continue across application requests or SDK instances. Because a live chat resides on one Query node, the SDK must retain enough routing state to continue the chat without tracking server-side lifecycle state.

# General Design

Conversational Query generates SQL++ from a natural-language prompt and, when requested, executes the generated statement using the caller's Couchbase identity. Model access uses separate model credentials.

A live chat resides in memory on one Query node. The SDK retains that node's logical identity for routing and serializes the client state needed to continue the chat as a `ChatHandle`. `attachChat(handle)` reconstructs a local `Chat` from that handle without server I/O.

PAUSE persists the chat in Query-managed storage and releases its live slot. RESUME restores a persisted chat to live server state.

This RFC does not define SDK APIs for chat discovery, model-provider discovery, typed EXPLAIN/ADVISE actions, a public chat-id accessor, single-entry Knowledge lookup, or changing a chat's inactivity timeout after BEGIN.

## API

```
Keyspaces = list<string>  // SQL++ keyspace paths

Cluster.conversationalQuery([credential ModelCredential], [model ModelOptions])
    -> ConversationalQuery

Cluster.knowledge() -> KnowledgeManager

ConversationalQuery
    query(prompt string,
          keyspaces Keyspaces,
          [options ConversationalQueryOptions])
        -> ConversationalQueryResult

    beginChat(keyspaces Keyspaces,
              [options BeginChatOptions])
        -> Chat

    attachChat(handle ChatHandle,
               [options AttachChatOptions])
        -> Chat                       // local operation; performs no server I/O

KnowledgeManager
    upsert(name string, keyspace string, value string)   // CREATE OR REPLACE
    create(name string, keyspace string, value string)   // CREATE; fails if it exists
    drop(name string, keyspace string)
    getAll([keyspace string]) -> KnowledgeEntry[]

KnowledgeEntry {
    name string
    keyspace string
    value string
}

Chat
    ask(prompt string, [options AskOptions])   -> ConversationalQueryResult
    pause([options PauseChatOptions])          -> ConversationalQueryMetaData
    resume()                                   -> ConversationalQueryMetaData
    end()                                      -> ConversationalQueryMetaData

    handle() -> ChatHandle

ConversationalQueryResult
    rows                                          // normal Query row handling
    metaData() -> ConversationalQueryMetaData
    generatedStatement() -> optional<string>

ConversationalQueryMetaData
    requestId() -> string
    clientContextId() -> string
    status() -> QueryStatus
    warnings() -> list<QueryWarning>
    metrics() -> optional<QueryMetrics>
    requestTokens() -> optional<TokenUsage>
    chatTokens() -> optional<TokenUsage>
    naturalLanguageProcessingTime() -> optional<Duration>

TokenUsage {
    promptTokens int
    completionTokens int
    totalTokens int
}

ModelCredential
    capella(email string, password string, organizationId string)
    credentialStore(credentialName string)
    apiKey(key string)

ModelOptions {
    provider string
    name string
    endpoint string
    region string
    outputTokenLimit int
    moderation optional<bool>
    raw map<string, JsonValue>
}

ConversationalQueryOptions {
    model ModelOptions
    credential ModelCredential
    hint string
    execute bool
    output ConversationalQueryOutput
    knowledge bool
}

BeginChatOptions {
    model ModelOptions
    credential ModelCredential
    inactivityTimeout Duration
    knowledge bool
    timeout Duration
}

AttachChatOptions {
    model ModelOptions
    credential ModelCredential
}

AskOptions {
    model ModelOptions
    credential ModelCredential
    hint string
    execute bool
    output ConversationalQueryOutput
}

Fields on both ConversationalQueryOptions and AskOptions {
    scanConsistency QueryScanConsistency
    consistentWith MutationState
    scanWait Duration
    useReplica bool
    clientContextId string
    profile QueryProfile
    metrics bool
    timeout Duration
    rawParameters map<string, JsonValue>
}

PauseChatOptions {
    model ModelOptions
    credential ModelCredential
    summarize optional<bool>
    timeout Duration
}

ConversationalQueryOutput {
    Sqlpp
    FtsSqlpp
    JsUdf
}
```

All option fields are optional. Only the Query options listed here are exposed by Conversational Query. `rawParameters` and `ModelOptions.raw` are distinct.

### Defaults and overrides

| Source | Effect |
| --- | --- |
| `conversationalQuery` | Default model and credential. |
| `query` options | Override for that request. |
| `beginChat` or `attachChat` options | Defaults for the returned `Chat`. |
| `ask` or `pause` options | Override for that operation. |
| `resume` or `end` | No model inputs. |

Supplying `ModelOptions` replaces the applicable default in full, including `raw`; fields are not merged. An unset `ModelOptions` inherits the applicable default, while an explicitly supplied empty `ModelOptions` does not.

Model and credential overrides are independent. Each override applies only to the request or returned `Chat` shown in the table and does not change the underlying defaults.

Model and credential defaults are not serialized in `ChatHandle`; they are request configuration and may change between turns. A `Chat` attached from a handle may therefore use different defaults from the `Chat` that created that handle.

### Ordinary Query controls

The listed Query controls retain their existing types, semantics, and wire mappings. The SDK must send any controls the caller explicitly sets.

`timeout` uses the existing Query timeout handling. A timeout reported by the SDK does not guarantee that Query has stopped processing the request.

BEGIN uses `BeginChatOptions.timeout` when set, and PAUSE uses `PauseChatOptions.timeout` when set. Otherwise, lifecycle operations use the cluster's default Query timeout.

`rawParameters` follows RFC 0056. The SDK must reject any raw parameter whose name, compared case-insensitively, begins with `natural`.

The SDK does not expose prepared statements, named or positional parameters, `readonly`, `preserveExpiry`, or `queryContext` for Conversational Query, and must send these requests as adhoc.

## Model Credentials

`ModelCredential` authenticates Query's calls to the model provider and is separate from Couchbase authentication. It is not accepted in `ClusterOptions`.

The **direct-provider path** sends model requests directly from Query to the configured provider instead of through the Capella iQ intermediary.

| Credential | Request parameters |
| --- | --- |
| `capella(email, password, organizationId)` | `natural_cred` = `email:password`; `natural_orgid` = organization id |
| `credentialStore(credentialName)` | `natural_config.cred_id` |
| `apiKey(key)` | `natural_config.api_key` |

The SDK sends the effective model credential on every request that may invoke the model. BEGIN, RESUME, and END carry no model credential.

The SDK must not fall back to another credential form after a failure.

If no model credential is set after resolving defaults and overrides, the request uses the direct-provider path.

The SDK must not add provider-specific authentication rules or additional credential kinds, and must not validate whether a model credential is valid for a particular provider.

For `capella`, the SDK sends the configured credentials to Query; the SDK does not authenticate directly with Capella.

The SDK must not inspect or modify cluster configuration to pre-validate model-credential prerequisites. Related failures use normal Query error handling.

Credentials are sent only as request parameters. The SDK must redact `natural_cred`, `natural_config.api_key`, and Capella passwords from errors, error contexts, diagnostics, logs, and tracing.

`credentialStore` sends the name of an existing server credential as `cred_id`; it does not send the stored secret.

## Model Options

`provider` and `name` are open strings. The SDK must not validate them against a hard-coded list of providers or models.

* `provider`: provider identifier. If omitted, the direct-provider path may use a server default.
* `name`: model identifier, sent as supplied. The SDK does not validate that the model exists.
* `endpoint`: complete direct-provider completions URL, sent as supplied. An explicit endpoint permits configurations without a model credential.
* `region`: provider region for the direct-provider path.
* `outputTokenLimit`: output-token limit for the direct-provider path.
* `moderation`: omit to use Query's default; preserve explicit `true` or `false`.
* `raw`: additional top-level `natural_config` entries; values may be any JSON value.

### Raw model options

`raw` is a forward-compatibility mechanism for additional Query `natural_config` fields. It is not passed through directly to the provider.

If a typed option and `raw` contain the same setting, the SDK must give the typed option precedence. An unset typed option must not overwrite a raw value.

The SDK must reject `cred_id` and `api_key` in `raw`. Credentials must use `ModelCredential`.

After applying typed-option precedence, the SDK copies the remaining raw entries to `natural_config` without filtering them.

After resolving defaults and overrides, the SDK must reject any supplied model option that is unsupported by the selected credential path. This validation applies only when the operation sends model inputs. See [Wire Mapping](#wire-mapping).

## One-shot Query

`query` submits a single natural-language prompt with at least one caller-supplied keyspace. It returns the generated SQL++ statement when present, together with any execution results.

Each call is independent. A one-shot query uses no chat state, creates no `ChatHandle`, and does not require routing to a particular Query node.

Execution is enabled by default. `execute(false)` requests generation without execution.

## Chat Lifecycle

A live chat exists only on its owning Query node and may be lost if that node becomes unavailable or the chat expires or is evicted. Unless the chat has been paused first, its state is not persisted.

A paused chat is persisted by Query and has a server-controlled expiry. The SDK must not predict expiry or reject requests using hard-coded limits.

Each Query node can hold only a limited number of live chats. PAUSE releases the slot used by the chat.

SDK implementations must support concurrent operations on a `Chat` without corrupting local routing state. Applications that require a particular order must serialize those operations themselves.

### Routing

The SDK retains routing information, not server lifecycle state. A retained owner is the Query node currently used as the chat's routing destination; it does not prove that the chat is still live there.

If no owner is retained, the SDK has no known Query node for ASK, PAUSE, or END. This does not mean the SDK knows the chat is paused.

Routing information may become stale if another SDK instance operates on the same chat, the chat expires, or the response to RESUME is lost.

| Event | SDK routing action |
| --- | --- |
| BEGIN succeeds on A | Retain A. |
| ASK succeeds, returns an error, or loses its response | Keep the retained owner. |
| PAUSE succeeds | Clear the retained owner. |
| PAUSE returns an error or loses its response | Keep the retained owner. |
| RESUME succeeds on B | Retain B. |
| RESUME response is lost after possible dispatch to B | Retain B. |
| RESUME fails before dispatch or returns an error | Keep the previous routing information. |
| END succeeds | Close this local wrapper. |
| END returns an error or loses its response | Keep the retained owner. |

When an operation keeps the routing information, it must not overwrite a routing change made by another concurrent operation. For example, a failed ASK must not restore an owner cleared by a concurrent PAUSE.

ASK, PAUSE, and END require a retained owner and are sent to that node. If no owner is retained, the SDK cannot route those operations. RESUME does not require one. See [Resume](#resume).

The SDK must not probe for chat state or perform lifecycle operations on the application's behalf. A server or transport error does not by itself block the application's next explicit operation.

If the retained owner cannot be resolved, the SDK must follow [Node Affinity](#node-affinity).

### Begin

`beginChat` creates a new live chat. The SDK sends BEGIN to any Query node with the requested keyspaces and optional inactivity timeout. The node that serves the request becomes the chat's owner.

As with `query`, the SDK must require at least one keyspace and reject an empty list with `InvalidArgumentException` (RFC 0058, #3).

The SDK must not treat a successful BEGIN as validation that those keyspaces exist; existence errors may surface on ASK.

The SDK does not send model options, model credentials, or the Knowledge choice with BEGIN.

The SDK retains the Knowledge choice for later ASK requests. Model and credential options on `beginChat` become defaults on the returned `Chat`.

### Ask

`ask` sends a prompt to the retained owner.

The chat's keyspaces are set by BEGIN and cannot change for an individual turn. The SDK sends the chat's Knowledge choice with every ASK. `execute` and `output` behave as they do for a one-shot `query`.

The SDK sends a hint only with the ASK where it is supplied; it does not become a default for later turns.

### Pause

`pause` saves the conversation so it can later be resumed and releases the chat's live slot on its Query node.

After a successful PAUSE, the SDK clears the retained owner.

`summarize` controls whether Query may summarize the conversation before saving it:

- unset: let Query decide;
- `true`: request summarization;
- `false`: do not summarize.

When `summarize` is unset or `true`, PAUSE may invoke the model. The SDK resolves the chat defaults with any PAUSE model and credential overrides and sends the resulting model inputs.

When `summarize` is `false`, PAUSE does not use the model. The SDK ignores any model or credential overrides, omits all model inputs, and does not validate the unused overrides.

If PAUSE returns an error or its response is lost, the SDK keeps the retained owner and does not infer whether PAUSE succeeded.

### Resume

`resume` restores a persisted chat to live server state. The retained owner is routing information and may be stale, so the SDK must not reject RESUME merely because an owner is retained.

If the retained owner resolves to a usable Query endpoint, the SDK sends RESUME to that node. Otherwise, it sends RESUME to any Query node.

RESUME is the one exception to the unresolvable owner rule under [Node Affinity](#node-affinity).

On success, the SDK records the node that served RESUME as the retained owner.

If the response is lost after the request may have been dispatched, the SDK records the destination node anyway, since that node is the owner if RESUME succeeded.

If RESUME fails before dispatch, the SDK keeps the previous routing information.

The SDK must not issue RESUME merely to probe chat state. A successful RESUME consumes the persisted chat and restores it as a live chat.

### End

`end` ends a live chat and releases its live slot. The SDK sends END to the retained owner.

END does not delete a paused chat. If no owner is retained, `end()` fails locally with `InvalidArgumentException` (RFC 0058, #3); the application must RESUME before END can be sent.

After END succeeds, the local `Chat` wrapper is closed. The SDK rejects further server operations, and `handle()` fails locally on that object.

Previously serialized handles cannot be revoked by closing this local `Chat`.

## Chat Handle

`ChatHandle` is an opaque, serializable representation of the client state needed to continue and route a chat. It does not represent server lifecycle state.

`attachChat(handle)` is local and must not issue RESUME. The returned `Chat` uses the attaching `ConversationalQuery` defaults unless `AttachChatOptions` replaces them. The handle contributes no model or credential defaults.

The handle must retain:

* the chat id;
* the Couchbase cluster's `clusterUUID`;
* the retained owner identity, or the absence of one;
* the chat's Knowledge choice.

A handle can be used only with its originating cluster. If the attaching `Cluster` already knows its `clusterUUID`, the SDK must reject a foreign handle immediately. Otherwise, it must validate the UUID before the first server operation.

`attachChat(handle)` must not perform I/O solely to obtain the UUID.

A serialized handle captures routing state that may later become stale. The SDK must not probe Query to validate or refresh it.

Each SDK must support serializing `ChatHandle` to JSON and reconstructing it from that JSON. The format is SDK-specific and must be versioned so incompatible handles can be rejected.

A handle must not contain Couchbase or model credentials, model configuration, a resolved Query endpoint, turn-specific hints, keyspaces, or the chat inactivity timeout.

## Knowledge

`KnowledgeManager` is obtained from `Cluster` and uses the cluster's Couchbase identity. Knowledge operations do not use model configuration, model credentials, or chat routing.

`getAll` returns the full text of every readable entry, optionally filtered by keyspace. It is not paginated.

`ConversationalQueryOptions.knowledge` controls one-shot use. `BeginChatOptions.knowledge` controls chat use. Both default to `false`.

For a chat, the SDK stores the Knowledge choice in the handle and sends the same value with every ASK. A summarizing PAUSE causes the model prompt to be rebuilt on the next ASK, and the SDK cannot know when such a rebuild is required.

# Implementation Details

## Node Affinity

A live chat is held in memory by one Query node, and Query does not route a chat request to its owner by chat id. The SDK must therefore send ASK, PAUSE, and END to the retained owner.

The SDK retains the owner's logical node identity, not a resolved Query endpoint. The identity is Query's canonical node name: the canonical hostname plus the non-TLS management port (`host:8091` by default), as reported by the `node` field of `system:natural_chats`, even when the SDK connects over TLS.

Before sending a chat operation to the retained owner, the SDK resolves that logical identity through normal topology handling for the selected network.

A retained owner is unresolvable when its logical node is absent from the SDK's current cluster configuration, or when that node has no usable Query endpoint for the selected network.

The SDK does not trigger an additional configuration refresh beyond normal topology handling.

For ASK, PAUSE, and END, an unresolvable owner causes `ServiceNotAvailableException` (RFC 0058, #4). The error context must include the chat id, retained owner identity, and operation.

The SDK keeps the routing information and does not automatically retry or reroute the request.

RESUME follows [its fallback rule](#resume).

A connection failure to a resolvable owner does not activate the RESUME fallback.

Transport failures use the normal RFC 0058 mappings: `UnambiguousTimeoutException` (#14) before dispatch, `AmbiguousTimeoutException` (#13) once the request may have been dispatched, and `RequestCanceledException` (#2) where applicable.

Any retry permitted by RFC 0049 must target the same Query node and stop once the request may have reached Query.

If no owner is retained, ASK, PAUSE, and END fail locally with `InvalidArgumentException` (RFC 0058, #3) because the SDK has no Query node to send them to.

The SDK must identify and record the node that served a successful BEGIN or RESUME before completing the operation. Query does not include the owner in its response.

Chat operations are unsupported through a proxy that hides the serving Query node. Alternate addresses are supported if serving-node attribution is preserved.

## Wire Mapping

The statements below encode Conversational Query options as a JSON object after `WITH`. For `USING AI`, the natural-language prompt follows that object as plain text. Ordinary Query controls exposed by `ConversationalQueryOptions` and `AskOptions` use their existing Query request-parameter mappings.

For example:

```text
USING AI WITH {"keyspaces":["travel-sample.inventory.hotel"],"knowledge":true,"execute":false,"output":"SQL"} Find hotels in Paris
```

### Operations

| Operation   | Statement                          | `WITH` fields                                                  | Model configuration and credential                            |
| ----------- | ---------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------- |
| `query`     | `USING AI WITH <options> <prompt>` | `keyspaces`, `knowledge`; optional `hint`, `execute`, `output` | Send the effective model configuration and credential.        |
| `beginChat` | `BEGIN CHAT WITH <options>`        | `keyspaces`, optional `timeout`                                | Omit.                                                         |
| `ask`       | `USING AI WITH <options> <prompt>` | Stored `knowledge`; optional `hint`, `execute`, `output`       | Send the effective model configuration and credential after applying operation overrides. |
| `pause`     | `PAUSE CHAT WITH <options>`        | Optional `summarize`                                           | Send the effective model configuration and credential after operation overrides when `summarize` is unset or `true`; omit model inputs when `false`. |
| `resume`    | `RESUME CHAT`                      | None                                                           | Omit.                                                         |
| `end`       | `END CHAT`                         | None                                                           | Omit.                                                         |

For every chat operation except BEGIN, the SDK must send the stored chat id as the string request parameter `natural_chatid`. It must not also include the chat id in the statement.

The SDK must use the `WITH` fields below for `hint`, `execute`, and `output`. It must not also send the corresponding `natural_*` request parameters.

### `WITH` Fields

| Field       | Encoding                                                                                                                                           |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `keyspaces` | Array of keyspace-path strings, for example `["travel-sample.inventory.hotel"]`. Sent only by `query` and BEGIN.                                   |
| `knowledge` | JSON boolean. Defaults to `false`. Send the one-shot value on `query` and the stored chat value on every `ask`.                                    |
| `hint`      | JSON string when supplied; otherwise omit.                                                                                                         |
| `execute`   | JSON boolean. Omit when unset; the server default is `true`.                                                                                        |
| `output`    | `Sqlpp` maps to `"SQL"`, `FtsSqlpp` to `"FTSSQL"`, and `JsUdf` to `"JSUDF"`. Omit when unset; the default is SQL output.                                             |
| `timeout`   | Inactivity timeout as a JSON integer number of seconds, separate from the Query request timeout; round a sub-second `Duration` up. Omit when unset. |
| `summarize` | JSON boolean when explicitly `true` or `false`; omit when unset.                                                                                   |

Keyspaces use SQL++ path notation. Pass `Keyspaces` entries verbatim. The Knowledge rules below do not apply.

### Model Options

Credential parameters are defined under [Model Credentials](#model-credentials).

| `ModelOptions`     | Capella path     | Direct-provider path                          |
| ------------------ | ---------------- | --------------------------------------------- |
| `provider`         | `natural_vendor` | `natural_config.provider`                     |
| `name`             | `natural_model`  | `natural_config.model`                        |
| `endpoint`         | Unsupported      | `natural_config.endpoint`                     |
| `region`           | Unsupported      | `natural_config.region`                       |
| `outputTokenLimit` | Unsupported      | `natural_config.output_token_limit`           |
| `moderation`       | Unsupported      | `natural_config.moderation`                   |
| `raw`              | Unsupported      | Additional top-level `natural_config` entries |

### Knowledge Statements

Knowledge operations use ordinary Query requests and normal Query routing. They do not use chat routing or model configuration.

For `create` and `upsert`, the SDK sends the Knowledge text as the named Query parameter `$value` rather than embedding it in the statement.

```text
create: CREATE KNOWLEDGE <name> FOR <keyspace> AS $value
upsert: CREATE OR REPLACE KNOWLEDGE <name> FOR <keyspace> AS $value
drop:   DROP KNOWLEDGE <name> FOR <keyspace>

getAll:
SELECT k.name, k.`bucket`, k.`scope`, k.`collection`, k.`value`
FROM system:natural_knowledge AS k
```

#### Name and keyspace encoding

Knowledge names and keyspaces are logical identifiers, not SQL++ expressions. The API accepts them in unquoted form.

The SDK must reject an empty Knowledge name with `InvalidArgumentException` (RFC 0058, #3). It must also reject any name or keyspace component whose first character is a backtick.

For accepted identifiers, the SDK wraps each identifier in backticks and escapes embedded backticks by doubling them.

`keyspace` is either a bucket name, such as `travel-sample`, or a three-part `bucket.scope.collection` path, such as `travel-sample.inventory.hotel`.

The SDK must reject empty components, any other number of components, and namespace prefixes. Dots are component separators, so this API cannot address a bucket whose name contains a dot.

For DDL, escape each keyspace component separately, for example:

```text
`travel-sample`.`inventory`.`hotel`
```

If the caller supplies only a bucket name, it refers to that bucket's `_default._default` collection, but the DDL contains only the escaped bucket name.

#### Listing entries

`getAll()` uses the projection above. Construct `KnowledgeEntry.keyspace` by joining `bucket`, `scope`, and `collection` with dots.

If the caller specifies only a bucket name, such as `travel-sample`, `getAll()` returns `travel-sample._default._default`. Both forms identify the same entry.

Any `KnowledgeEntry.keyspace` returned by `getAll()` must be accepted unchanged by other `KnowledgeManager` operations. The only exception is a keyspace whose bucket name contains a dot, which this string form cannot represent unambiguously.

When `getAll(keyspace)` includes a keyspace filter, add:

```text
WHERE k.`bucket` = $bucket
  AND k.`scope` = $scope
  AND k.`collection` = $collection
```

When filtering `getAll()` by keyspace, the SDK binds the bucket, scope, and collection as separate Query parameters. If the caller supplies only a bucket name, the SDK uses `_default` for the scope and collection.

#### Create and drop behavior

`create` must not add `IF NOT EXISTS`, and `drop` must not add `IF EXISTS`. Failures use ordinary Query error handling; this RFC adds no typed exception.

Other server-supported Knowledge grammar is outside this SDK API.

## Results

`ConversationalQueryResult` is a separate public result type with normal Query row semantics and the metadata defined below. SDKs may reuse existing Query implementation and types internally; the result type does not need to inherit from `QueryResult`.

### Generated statement

When present, the top-level `generated_statement` field contains the generated SQL++ statement. The SDK exposes it through the optional `generatedStatement()` accessor.

The generated statement must be available without iterating over result rows.

When rows are present, `generated_statement` precedes `results`, but other metadata may precede it. The SDK must not depend on a fixed overall field order.

The SDK must accept responses without `results` or `signature`, including generation without execution and lifecycle responses.

### Chat metadata

BEGIN returns a top-level string `chatId`. The SDK must store it with the serving node before returning the `Chat`.

### Token usage

`requestTokens` and `chatTokens` are optional top-level response fields that map to `ConversationalQueryMetaData.requestTokens()` and `chatTokens()`. `requestTokens` reports token usage for the current request.

`chatTokens` reports cumulative token usage for the chat and, when present, is also exposed in metadata returned by PAUSE, RESUME, and END.

When present, each token object contains `promptTokens`, `completionTokens`, and `totalTokens`. A counter not reported by the provider appears as `0`. Query omits the object when all three counters are `0`.

Token usage is not reported on the Capella path.

The SDK must preserve the returned counters exactly. It must not synthesize an all-zero `TokenUsage` when the object is absent, derive `totalTokens` from the other counters, or reject a response because the counters do not add up.

Conversational Query reuses the existing `QueryMetrics` type unchanged. The SDK exposes `metrics.naturalLanguageProcessingTime` as `ConversationalQueryMetaData.naturalLanguageProcessingTime()` rather than adding it to `QueryMetrics`.

### Execution

`execute(true)` requests execution, but a successful response does not guarantee that the generated SQL++ was executed.

`JsUdf` generates a JavaScript UDF as `CREATE FUNCTION`. Creating it requires a separate ordinary Query request.

The response has no explicit execution-status field ([MB-73787](https://jira.issues.couchbase.com/browse/MB-73787)), so the SDK must not infer whether execution occurred from the presence of a generated statement, result rows, metrics, or a `signature`.

## Errors

Except for the rules below, normal Query error handling (RFC 0058) applies.

The SDK must preserve the server code, message, and `reason` in the Query error context, subject to credential redaction. `reason` may be a nested Query error with its own `code` or a plain string.

Invalid arguments, closed wrappers, and incompatible or foreign-cluster handles raise `InvalidArgumentException` (RFC 0058, #3), unless this RFC names another error.

| Code  | SDK handling                                                                                                                                     |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| 19220 | `RateLimitedException` (RFC 0058, #21). |
| 19221 | `AmbiguousTimeoutException` (RFC 0058, #13). Query may already have called the model. |
| 19222 | Natural-language processing is disabled on the receiving node. `FeatureNotAvailableException` (RFC 0058, #15). |
| 19236 | The receiving node has no live chat with that id. Preserve as a Query error; do not clear routing or infer lifecycle. |
| 19240 | The node has no available live-chat slots. `QuotaLimitedException` (RFC 0058, #22), including when nested in a 19243 `reason`; preserve both codes. |
| 19241 | The Couchbase identity failed the chat ownership check. `AuthenticationFailureException` (RFC 0058, #6). |
| 19243 | Preserve as a Query error, except for nested 19240. It does not prove that the persisted chat is missing. |
| 19255 | RESUME found the chat already live on the receiving node. Preserve as a Query error; do not infer a lifecycle transition. |

## Retries

The SDK must not automatically retry a `USING AI` request or chat lifecycle request once it may have reached Query. Knowledge operations continue to use normal Query retry handling.

A failed request may already have called the model, changed chat history, or executed generated SQL++. Replaying it may therefore repeat work or state changes. This remains true with `execute(false)`, since model calls or chat changes may already have occurred.

The SDK may use normal RFC 0049 retry handling only when the request is known not to have reached Query. A retry of a chat operation must target the same Query node.

A pre-dispatch retry of BEGIN, or of RESUME when there is no usable retained owner, may choose a different node.

SDKs must not add a generic retry loop for model-provider failures.

The SDK must preserve the server's `retry` flag in error context when present. That flag does not authorize replay of the enclosing Conversational Query request.

## Feature Detection

Chat support is advertised by `conversationalQuery` in `clusterCapabilities.n1ql`. If that capability is absent, `beginChat` and any `Chat` operation that would contact Query must fail locally with `FeatureNotAvailableException`.

The capability check does not apply to local operations (`attachChat`, handle parsing, or handle serialization), one-shot `query`, or Knowledge operations. Unsupported Knowledge operations use normal Query error handling.

The SDK must not add a separate version check for the direct-provider path.

The advertised capability does not guarantee that every Query node will accept natural-language requests. If the retained owner returns 19222, the SDK returns the error without rerouting; the owner remains the routing destination, so the RESUME fallback does not apply.

# Documentation

SDK documentation should explain `ChatHandle` persistence, stale routing information, lost responses, chat expiry and live-chat limits, and generation without execution. It should tell applications to save a new handle after routing changes and to serialize lifecycle operations when their order matters.

Documentation should warn against using Knowledge to store secrets and describe the Couchbase authorization boundary and what request, chat, and Knowledge content may be sent to model providers. It should link to authoritative server documentation for Capella authentication, credential-store prerequisites, direct-provider configuration, and provider-specific data handling.


# Changelog

* Revision 3 - 2026-09-10 (by Anirudh Lakhotia)

  * Reworked the draft to follow the SDK RFC structure and consolidated the proposed public API.
  * Added Knowledge management, feature detection, and the `Indeterminate` PAUSE state.
  * Consolidated model credential and option handling.
  * Removed superseded background, requirements, questions, deployment prerequisites, and wire-example sections.

* Revision 2 - 2026-09-02 (by Jared Casey)
  * Expanded the original draft with chat lifecycle, credential types, and result handling.

* Revision 1 - 2026-08-31 (by Jared Casey)
  * Original draft.

# Signoff

| Language | Team Member | Signoff Date | Revision |
|---|---|---|---|
| .NET | | | |
| C++ | | | |
| Go | | | |
| Java | | | |
| Kotlin | | | |
| Node.js | | | |
| PHP | | | |
| Python | | | |
| Ruby | | | |
| Scala | | | |
