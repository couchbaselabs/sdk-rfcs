# Meta

* RFC Name: Conversational Query
* RFC ID: 86
* Start Date: 2026-08-31
* Owner: Anirudh Lakhotia, Jared Casey
* Current Status: DRAFT
* Revision: 3
* API Stability: Volatile
* Supporting Material: [sdk-design](https://github.com/couchbaselabs/sdk-design/tree/main/server-aligned/totoro/query-begin-chat)

# Summary

Conversational Query lets applications submit natural-language prompts to the Query service and receive generated statements and, when executed, query results. This RFC defines the SDK APIs for one-shot queries, multi-turn chats, model configuration and credentials, and continuing chats across application requests.

# Motivation

A live chat resides on a single Query node, and Query does not forward chat requests to its owner. Successive requests may be handled by different SDK instances, so applications need a way to continue a chat without retaining the original SDK object. The SDK therefore provides explicit lifecycle operations and a serializable handle for routing subsequent requests, while Query remains responsible for the chat's server-side state.

# General Design

One-shot queries are independent requests. Chats use explicit BEGIN, ASK, PAUSE, RESUME, and END operations. Query manages chat state and persists it while paused; the SDK retains the chat id and routing information needed to reach the owning node.

Model configuration and credentials are supplied per request or as defaults for a `Chat`, rather than stored in `ChatHandle`. Query generates SQL++ and executes eligible statements by default, subject to the `execute` option. Generated statements run under the caller's Couchbase identity, while model-provider access uses separate credentials.

## API

```
Keyspaces = list<string>  // SQL++ keyspace paths

Cluster.conversationalQuery() -> ConversationalQuery

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

Chat
    ask(prompt string, [options AskOptions])   -> ConversationalQueryResult
    pause([options PauseOptions])              -> ConversationalQueryMetaData
    resume([options ResumeOptions])            -> ConversationalQueryMetaData
    end([options EndOptions])                  -> ConversationalQueryMetaData

    handle() -> ChatHandle

ChatHandle
    toJson() -> string
    fromJson(json string) -> ChatHandle   // static; local operation

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
    generatedStatementExecuted() -> optional<bool>

TokenUsage {
    promptTokens optional<int>
    completionTokens optional<int>
    totalTokens optional<int>
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
    execute optional<bool>
    output ConversationalQueryOutput
    useKnowledge bool
    queryContext string
}

BeginChatOptions {
    model ModelOptions
    credential ModelCredential
    inactivityTimeout Duration
    useKnowledge bool
    queryContext string
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
    execute optional<bool>
    output ConversationalQueryOutput
}

Fields shared by ConversationalQueryOptions and AskOptions {
    scanConsistency QueryScanConsistency
    consistentWith MutationState
    scanWait Duration
    useReplica bool
    clientContextId string
    metrics bool
    timeout Duration
    rawParameters map<string, JsonValue>
}

PauseOptions {
    model ModelOptions
    credential ModelCredential
    summarize optional<bool>
    timeout Duration
}

ResumeOptions {
    timeout Duration
}

EndOptions {
    timeout Duration
}

ConversationalQueryOutput {
    Sqlpp
    FtsSqlpp
    JsUdf
}
```

All option fields are optional. Fields declared `optional<T>` must distinguish unset from an explicit value. For other Conversational Query-specific option fields, empty strings, numeric zero, and empty maps are treated as unset. Unless this RFC specifies otherwise, unset Conversational Query-specific fields are omitted from requests. Normal Query defaults and generated parameters still apply to every request the SDK sends to Query, including `metrics: false` and `client_context_id`.

Existing SDK controls for timeouts, retries, tracing, and cancellation also apply, subject to the rules in this RFC. `rawParameters` supplies Query request parameters, while `ModelOptions.raw` supplies additional model configuration.

### Defaults and overrides

| Source | Effect |
| --- | --- |
| `query` options | Model and credential for that request. |
| `beginChat` or `attachChat` options | Defaults for the returned `Chat`. |
| `ask` or `pause` options | Override for that operation. |
| `resume` or `end` | No model inputs. |

When an operation sends model inputs, a supplied `ModelOptions` replaces the default model configuration, including `raw`; the SDK does not merge individual fields. If `ModelOptions` is omitted, the default is used. If an empty `ModelOptions` is explicitly supplied, it replaces the default.

Model and credential overrides are independent: overriding the model does not change the credential, and vice versa. Overrides on `ask` and `pause` apply only to that operation and do not change the `Chat`'s defaults. When model inputs are sent, an operation without a credential override uses the `Chat`'s default credential. There is no per-operation option to clear that default.

A `Chat` captures its defaults when `beginChat` or `attachChat` is called; later changes to the supplied option objects do not affect it.

### Ordinary Query controls

The listed Query controls retain their existing types, semantics, and wire mappings.

A timeout reported by the SDK does not guarantee that Query has stopped processing the request.

BEGIN, PAUSE, RESUME, and END use their operation's `timeout` when set. Otherwise, the SDK's normal Query operation timeout applies.

`rawParameters` follows RFC 0056. The SDK must reject raw parameters whose names begin with `natural` (case-insensitively), to avoid conflicts with Conversational Query request parameters.

The SDK does not expose prepared statements, named or positional parameters, `readonly`, or `preserveExpiry` for Conversational Query, and must send these requests as adhoc.

`queryContext` uses the ordinary Query `query_context` request parameter and is sent unchanged. The SDK does not parse, quote, or qualify it. It is sent only by one-shot `query` and `beginChat`. For a chat, Query uses it to resolve the keyspaces supplied to BEGIN. It is not retained as chat configuration or in the `ChatHandle`.

## Model Credentials

`ModelCredential` authenticates Query's calls to the model provider and is separate from Couchbase authentication. It is not accepted in `ClusterOptions`.

The **direct-provider path** sends model requests directly from Query to the configured provider instead of through the Capella iQ intermediary. The Capella path is selected when the effective credential is `capella`. `credentialStore`, `apiKey`, or no credential use the direct-provider mapping.

| Credential | Request parameters |
| --- | --- |
| `capella(email, password, organizationId)` | `natural_cred` = `email:password`; `natural_orgid` = organization id |
| `credentialStore(credentialName)` | `natural_config.cred_id` |
| `apiKey(key)` | `natural_config.api_key` |

The SDK must not fall back to another credential form after a failure.

The SDK must not add provider-specific authentication rules or additional credential kinds, and must not validate whether a model credential is valid for a particular provider.

The SDK must not inspect or modify cluster configuration to pre-validate model-credential prerequisites. Related failures use normal Query error handling.

Credentials are sent only as request parameters. The SDK must redact `natural_cred` and `natural_config` from errors, error contexts, diagnostics, logs, and tracing. Provider and model names may be reported separately. Capella passwords, API keys, and `ModelOptions.raw` values must not appear in diagnostic output, including string representations of credentials and options.

## Model Options

`provider` and `name` are open strings, and `endpoint` is a provider-specific URL; all are sent as supplied. The SDK must not validate `provider` or `name` against a hard-coded list of providers or models, or check that the model exists. An explicit endpoint can be used without a model credential. `raw` values may be any JSON value.

### Raw model options

`raw` is a forward-compatibility mechanism for additional Query `natural_config` fields.

If a supplied typed option and a `raw` entry map to the same `natural_config` key, the SDK must give the typed option precedence. An unset typed option must not overwrite a raw value.

The SDK must reject `cred_id` and `api_key` in `raw`, compared case-insensitively. Credentials must use `ModelCredential`. The SDK copies the remaining `raw` entries to `natural_config` without filtering them.

After resolving defaults and overrides, the SDK must reject any supplied model option that is unsupported by the selected credential path. This check and the `cred_id`/`api_key` check above apply only when an operation sends model inputs, and only to the resolved options. `beginChat` and `attachChat` capture defaults without validating them. See [Model option mapping](#model-option-mapping).

## One-shot Query

`query` submits a single natural-language prompt with at least one caller-supplied keyspace. It returns the generated statement when present, together with any execution results. The SDK must reject an empty keyspace list with `InvalidArgumentException` (RFC 0058, #3).

## Chat Lifecycle

A live chat exists only on its owning Query node and may be lost if that node becomes unavailable or the chat expires. Query persists chat state only while the chat is paused, and a paused chat has a server-controlled expiry. The SDK must not predict that expiry or reject requests using hard-coded limits.

SDK implementations must support concurrent operations on a `Chat` without corrupting local state, but do not guarantee their order. Applications must serialize PAUSE, RESUME, and END operations on the same chat, including across `Chat` instances, and any other operations whose order matters.

### Routing

The SDK retains routing information, not server lifecycle state. A retained owner identifies the Query node to which chat operations are routed; it does not guarantee that the chat is still live on that node.

Routing information may become stale if another SDK instance operates on the same chat, the chat expires, or the response to RESUME is lost.

The SDK updates the retained owner as follows:

| Operation and outcome                                                   | SDK action                                   |
| ----------------------------------------------------------------------- | -------------------------------------------- |
| BEGIN succeeds                                                          | Retain the Query node BEGIN was sent to.     |
| ASK succeeds, fails, or loses its response                              | Keep the retained owner.                     |
| PAUSE succeeds                                                          | Clear the retained owner.                    |
| PAUSE fails or loses its response                                       | Keep the retained owner.                     |
| RESUME succeeds                                                         | Retain the Query node RESUME was sent to.    |
| RESUME may have been dispatched, but its outcome is unknown             | Retain the Query node RESUME was sent to.    |
| RESUME returns 19255                                                    | Retain the Query node RESUME was sent to.    |
| RESUME to the retained owner returns 19243 as its first top-level error | Clear the retained owner.                    |
| RESUME fails before dispatch or returns any other error                 | Keep the previous routing information.       |
| END succeeds                                                            | Close the local `Chat` wrapper.              |
| END fails or loses its response                                         | Keep the retained owner.                     |

The SDK must not probe for chat state or perform lifecycle operations on the application's behalf. A server or transport error does not by itself prevent the application's next explicit operation.

### Begin

`beginChat` creates a new chat. The SDK selects a Query node (see [Node Affinity](#node-affinity)) and sends BEGIN to that node with the requested keyspaces, Knowledge choice, and optional inactivity timeout.

The keyspace rule for `query` also applies to `beginChat`.

### Ask

`ask` sends the prompt to the retained owner.

### Pause

`pause` sends PAUSE to the retained owner.

`summarize` controls whether Query may summarize the conversation before saving it. When it is unset or `true`, PAUSE may invoke the model. When `summarize` is `false`, the SDK omits model inputs and does not validate unused model or credential overrides.

### Resume

The SDK must allow `resume` even when a retained owner exists, since that routing information may be stale.

If the retained owner is resolvable, the SDK sends RESUME to that node. Otherwise, it selects a Query node as for BEGIN.

### End

`end` sends END to the retained owner.

After END succeeds, the local `Chat` wrapper is closed. Further server operations and `handle()` on that object fail locally with `InvalidArgumentException` (RFC 0058, #3).

## Chat Handle

`ChatHandle` is an opaque, serializable representation of the client state needed to continue and route a chat.

`handle()` returns an immutable snapshot of the `Chat`'s current handle state.

`attachChat(handle)` is local and must not issue RESUME. The returned `Chat` takes its model and credential defaults only from `AttachChatOptions`.

The handle's JSON form is an object with only these fields:

| Field | Value |
| --- | --- |
| `chatId` | The chat id, as a string. |
| `owner` | The retained owner's `nodeUUID` (see [Node Affinity](#node-affinity)), or `null` when no owner is retained. |

A `ChatHandle` must be used with the same Couchbase cluster where the chat was created. The SDK does not verify this.

Each SDK must support JSON serialization and reconstruction of `ChatHandle` through `toJson()` and `fromJson()` or language-idiomatic equivalents. `fromJson()` must reject a malformed handle, a handle missing any field above, or a field of the wrong type (`owner` may be `null`) with `InvalidArgumentException`, and must ignore unrecognized fields.

A `ChatHandle` must not contain Couchbase or model credentials or model configuration.

## Knowledge

Conversational Query can use Knowledge entries configured in Query. `ConversationalQueryOptions.useKnowledge` controls one-shot use. `BeginChatOptions.useKnowledge` sets chat use at BEGIN, and Query retains it for the chat. Both default to `false`.

# Implementation Details

## Node Affinity

A live chat is held in memory by one Query node, and Query does not forward a chat request to its owner. The SDK must therefore send ASK, PAUSE, and END to the retained owner.

The SDK retains the owner's `nodeUUID` from `nodesExt` in the cluster configuration.

For operations routed to a retained owner, the SDK resolves that owner's `nodeUUID` against the current cluster configuration before each dispatch attempt and uses the node's Query endpoint for the selected network and TLS setting.

A retained owner is unresolvable if its `nodeUUID` is absent from the current cluster configuration or its node has no usable Query endpoint for the selected network and TLS setting. Before the first cluster configuration arrives, the SDK follows normal Query service readiness handling instead of treating the owner as unresolvable. Transient connection state, including a local circuit breaker, does not make a resolvable owner unresolvable.

The SDK does not trigger an additional configuration refresh beyond normal topology handling.

For ASK, PAUSE, and END, no retained owner or an unresolvable owner causes `ChatCurrentlyUnavailableException`.

The SDK fails locally with this error, keeps the routing information, and does not automatically retry or reroute the request.

A connection failure to a resolvable owner must not cause RESUME to select another node.

Transport failures follow the normal SDK error mappings (RFC 0058). Timeouts before dispatch map to `UnambiguousTimeoutException` (#14); timeouts after possible dispatch map to `AmbiguousTimeoutException` (#13). Cancellation maps to `RequestCanceledException` (#2) where applicable.

`RequestCanceledException` does not by itself establish that the request was not dispatched. If the SDK knows cancellation occurred before dispatch, the pre-dispatch rules apply. Otherwise, the outcome is unknown and the SDK must not automatically retry or infer chat state.

A complete, successfully parsed Query response is treated as a returned server response for routing, even if its error maps to `AmbiguousTimeoutException`. If no complete response is available after the request may have been dispatched, the outcome is unknown.

For BEGIN, or RESUME without a resolvable retained owner, the SDK selects a Query node with a `nodeUUID` and a usable Query endpoint for the selected network and TLS setting in its current cluster configuration, and sends the request there. The SDK must record the owner before completing a successful BEGIN or RESUME.

Chats are not supported on Capella Private Link. Over `couchbase2://`, or when the current cluster configuration maps different Query `nodeUUID`s to the same Query host and port for the selected network and TLS setting, `beginChat` and any `Chat` operation that would contact Query must fail locally with `FeatureNotAvailableException`.

## Wire Mapping

Statement options are encoded as a JSON object in the `WITH` clause. For `USING AI`, the natural-language prompt follows the object as plain text.

For example:

```text
USING AI WITH {"keyspaces":["travel-sample.inventory.hotel"],"knowledge":true,"execute":false,"output":"SQL"} Find hotels in Paris
```

### Operations

| Operation   | Statement                          | `WITH` fields                                                  | Model inputs                                                                    |
| ----------- | ---------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `query`     | `USING AI WITH <options> <prompt>` | `keyspaces`, `knowledge`; optional `hint`, `execute`, `output` | Send the supplied model and credential, if any.                                 |
| `beginChat` | `BEGIN CHAT WITH <options>`        | `keyspaces`, `knowledge`; optional `timeout`                   | Do not send; supplied values become `Chat` defaults.                            |
| `ask`       | `USING AI WITH <options> <prompt>` | Optional `hint`, `execute`, `output`                           | Send the effective model and credential, if any.                                |
| `pause`     | `PAUSE CHAT [WITH <options>]`      | Optional `summarize`                                           | Send the effective model and credential, if any, unless `summarize` is `false`. |
| `resume`    | `RESUME CHAT`                      | None                                                           | None.                                                                           |
| `end`       | `END CHAT`                         | None                                                           | None.                                                                           |

`USING AI` always includes `WITH`, using `WITH {}` when there are no fields, so that Query does not parse prompt text as options. Other statements omit `WITH` when there are no fields.

For every chat operation except BEGIN, the SDK must send the stored chat id as the string request parameter `natural_chatid`. It must not also include the chat id in the statement.

The SDK must use the `WITH` fields below for `hint`, `execute`, and `output`. It must not also send the corresponding `natural_*` request parameters.

### `WITH` Fields

| Field       | Encoding                                                                                                                                           |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `keyspaces` | Array of SQL++ keyspace-path strings, passed verbatim, for example `["travel-sample.inventory.hotel"]`.                                           |
| `knowledge` | `useKnowledge` as a JSON boolean, defaulting to `false`. Send on `query` and `beginChat`; omit on `ask`. |
| `hint`      | JSON string when supplied; otherwise omit.                                                                                                         |
| `execute`   | JSON boolean. Omit when unset; the server default is `true`.                                                                                        |
| `output`    | `Sqlpp` maps to `"SQL"`, `FtsSqlpp` to `"FTSSQL"`, and `JsUdf` to `"JSUDF"`. Omit when unset; the default is SQL output.                                             |
| `timeout`   | `BeginChatOptions.inactivityTimeout`, encoded as an integer number of seconds, rounded up. Omit when unset. |
| `summarize` | JSON boolean when explicitly `true` or `false`; omit when unset.                                                                                   |

### Model option mapping

Credential parameters are defined under [Model Credentials](#model-credentials). The SDK sends `natural_config` as a JSON object and omits it when it would be empty.

| `ModelOptions`     | Capella path     | Direct-provider path                          |
| ------------------ | ---------------- | --------------------------------------------- |
| `provider`         | `natural_vendor` | `natural_config.provider`                     |
| `name`             | `natural_model`  | `natural_config.model`                        |
| `endpoint`         | Unsupported      | `natural_config.endpoint`                     |
| `region`           | Unsupported      | `natural_config.region`                       |
| `outputTokenLimit` | Unsupported      | `natural_config.output_token_limit`           |
| `moderation`       | Unsupported      | `natural_config.moderation`                   |
| `raw`              | Unsupported      | Additional top-level `natural_config` entries |

## Results

`ConversationalQueryResult` is a separate public result type with normal Query row semantics and the metadata defined below.

### Generated statement

When present, the top-level `generated_statement` field contains the generated statement. The SDK exposes it through the optional `generatedStatement()` accessor.

The generated statement must be available without iterating over result rows.

If both `generated_statement` and `results` are present, `generated_statement` precedes `results`. The SDK must not depend on a fixed order for the other fields.

The SDK must accept responses without `results` or `signature`, including generation without execution and lifecycle responses.

### Chat ID

The SDK reads the top-level string `chatId` from the BEGIN response and retains it in the returned `Chat`.

### Token usage

`requestTokens` and `chatTokens` are optional top-level response fields that map to `ConversationalQueryMetaData.requestTokens()` and `chatTokens()`. `requestTokens` reports token usage for the current request. The recognized token counter fields are `promptTokens`, `completionTokens`, and `totalTokens`.

`chatTokens` reports cumulative token usage for the chat and, when present, is also exposed in metadata returned by PAUSE, RESUME, and END.

Each token counter is optional. If a counter is absent from the Query response, its accessor is absent.

The SDK must preserve the returned counters exactly. It must not synthesize an all-zero `TokenUsage` when the object is absent, derive `totalTokens` from the other counters, or reject a response because the counters do not add up.

### Metrics

The SDK exposes `metrics.naturalLanguageProcessingTime` through `ConversationalQueryMetaData.naturalLanguageProcessingTime()`, converting Query's duration string to a `Duration`. The accessor is absent when the field is not returned. Existing shared Query types, including `QueryMetrics`, remain unchanged.

### Execution

When present, `metrics.generatedStatementExecuted` is exposed through `ConversationalQueryMetaData.generatedStatementExecuted()`. The accessor is absent when the field is not returned. The SDK must not infer execution status from the generated statement, result rows, or other metadata.

## Errors

Except for the rules below, normal Query error handling (RFC 0058) applies.

Unless this RFC specifies another error, invalid arguments raise `InvalidArgumentException` (RFC 0058, #3).

The SDK must preserve the server code, message, and `reason` in the Query error context, subject to credential redaction. `reason` may be a nested Query error with its own `code` or a plain string.

The mappings below apply only to Conversational Query operations. When a response contains multiple Query errors, the first top-level Query error determines both the SDK error mapping and any error-specific routing rule. Nested errors affect handling only where this RFC explicitly says so.

| Code  | SDK handling                                                                                                                                     |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1080  | For all Conversational Query operations, map 1080 to `AmbiguousTimeoutException` (RFC 0058, #13), unlike ordinary Query handling. The operation may have taken effect before the timeout. |
| 19220 | Too many natural-language requests are waiting on the receiving node. `RateLimitedException` (RFC 0058, #21). |
| 19221 | A natural-language request waited too long for admission on the receiving node. `RateLimitedException` (RFC 0058, #21). |
| 19222 | Natural-language processing is disabled on the receiving node. `FeatureNotAvailableException` (RFC 0058, #15). |
| 19236 | On ASK, PAUSE, or END, the receiving node has no live chat with that id. `ChatCurrentlyUnavailableException`. |
| 19240 | The node has no available live-chat slots. `TemporaryFailureException` (RFC 0058, #7), including when nested in a 19243 `reason`; preserve both codes. |
| 19241 | The Couchbase identity failed the chat ownership check. `AuthenticationFailureException` (RFC 0058, #6). |
| 19243 | RESUME could not restore the chat on the receiving node. Except for nested 19240, no new typed exception is defined for this code: use the SDK's existing handling for an unmapped Query error and preserve the server code and error context. It does not prove that the persisted chat is missing. Routing follows [Routing](#routing). |
| 19255 | RESUME found the chat already live on the receiving node. No new typed exception is defined for this code: use the SDK's existing handling for an unmapped Query error, and preserve the server code and error context. Routing follows [Routing](#routing). |

`ChatCurrentlyUnavailableException` is new for this feature and derives directly from `CouchbaseException`. It is raised for ASK, PAUSE, and END when no owner is retained, the retained owner is unresolvable, or Query returns error 19236. Routing information is retained. The error does not establish that the chat has ended, expired, or lacks persisted state.

The `ChatCurrentlyUnavailableException` error context includes the chat id, the retained owner's `nodeUUID` when one is retained, and the operation.

## Retries

The SDK must not automatically retry a `USING AI` request or chat lifecycle request once it may have reached Query. This rule and the affinity rules in this RFC take precedence over any user-configured retry strategy.

This applies even with `execute(false)`.

The SDK may use normal RFC 0049 retry handling only when the request is known not to have reached Query. A retry of an operation sent to a retained owner must target that same Query node.

A pre-dispatch retry of BEGIN, or of RESUME when there is no resolvable retained owner, may choose a different node.

The SDK must preserve the server's `retry` flag in error context when present. That flag does not authorize replay of the enclosing Conversational Query request.

## Tracing

Each server-facing operation uses its own outer request span name, following RFC 0067:

| Operation | Span name |
| --- | --- |
| `query` | `conversational_query` |
| `beginChat` | `conversational_query_begin_chat` |
| `ask` | `conversational_query_ask` |
| `pause` | `conversational_query_pause` |
| `resume` | `conversational_query_resume` |
| `end` | `conversational_query_end` |

## Feature Detection

Chat support is advertised by `conversationalQuery` in `clusterCapabilities.n1ql`. If that capability is absent, `beginChat` and any `Chat` operation that would contact Query must fail locally with `FeatureNotAvailableException`. Normal Query service readiness handling applies until capability information is available. If the capability is known to be absent, `FeatureNotAvailableException` takes precedence over `ChatCurrentlyUnavailableException` for a missing or unresolvable retained owner.

The capability check does not apply to local operations (`attachChat`, handle parsing, or handle serialization) or to one-shot `query`.

Over `couchbase2://`, `ConversationalQuery.query` must fail locally with `FeatureNotAvailableException`.

The SDK must not add a separate version check for the direct-provider path.

# Documentation

SDK documentation should explain how to save and restore a `ChatHandle`. A handle excludes model and credential configuration. It is a snapshot, so applications should save a new handle after `beginChat`, `pause`, or `resume`. Model and credential options supplied to `beginChat` or `attachChat` become defaults for the returned `Chat`; those supplied to `beginChat` are not sent with BEGIN.

Documentation must explain that applications must serialize PAUSE, RESUME, and END operations on the same chat, including across `Chat` instances, and any other operations whose order matters. It should explain that an explicit `resume()` may recover from `ChatCurrentlyUnavailableException`, but ambiguous lifecycle outcomes must not be blindly replayed. END does not delete a paused chat, so the application must resume it first. After END succeeds, previously serialized handles can still be attached, but they cannot continue the chat.

Documentation should explain that `execute(true)` does not guarantee execution. Applications that need to determine whether a statement was executed should enable metrics and inspect `generatedStatementExecuted()` when present. Creating a function generated with `JsUdf` requires a separate ordinary Query request.

SDK documentation must cover the unsupported connection types listed under [Node Affinity](#node-affinity) and [Feature Detection](#feature-detection), and link to the server prerequisites for PAUSE and RESUME, including edition and metadata-storage requirements.

Documentation should warn against storing secrets in Knowledge and explain the Couchbase authorization boundary and what request, chat, and Knowledge content may be sent to model providers. It should state that Knowledge entries are managed through ordinary Query statements and link to server documentation for Capella authentication, credential-store prerequisites, direct-provider configuration, and provider-specific data handling.


# Changelog

* Revision 3 - 2026-09-10 (by Anirudh Lakhotia)

  * Reworked the draft to follow the SDK RFC structure and consolidated the proposed public API.
  * Added feature detection.
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
