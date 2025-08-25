# Meta

- RFC Name: SDK3 Bootstrapping
- RFC ID: 0048-sdk3-bootstrapping
- Start Date: April 1, 2019
- Owner: Brett Lawson \<brett@couchbase.com\>
- Current Status: ACCEPTED
- Revision: #4
- Relates To
  - [SDK-RFC#7 (Cluster Level Authentication)][sdk-rfc-0007]
  - [SDK-RFC#16 (RBAC)][sdk-rfc-0016]
  - [SDK-RFC#24 (Fast-Failover)][sdk-rfc-0024]
  - [SDK-RFC#35 (Client Connection IDs)][sdk-rfc-0035]
  - [SDK-RFC#47 (Unified User Agent)][sdk-rfc-0047]
- [Original Google Drive Doc](https://docs.google.com/document/d/1SUSBM9XoTnpaeew0bq4ABmgISc76fjzrY2vapZ10XD4)

# Motivation

The existing SDKs currently do not have a well-defined process for connecting to a cluster or bucket.  This document aims to resolve a number of dissimilarities between the SDK bootstrapping process as well as aims to resolve a number of deficiencies in the current best-practice standards for performing bootstrapping through the use of the newly introduced G3CP feature in Mad Hatter.  The existing SDKs do not currently support performing cluster level queries natively.  Instead they are implemented through a maneuver where the SDK locates the memcached connection for an arbitrary open bucket, and utilizes the configuration fetched for that bucket to deduce the cluster-level configurations.  Starting in Mad Hatter, the server will support performing a configuration fetch on a memcached connection which has not yet selected a bucket.

# Summary

This document describes a number of different behaviours within a client.  We first start by discussing how individual memcached connections should be established and authenticated.  We then further describe the overall connection behaviour for an entire cluster, the process for connecting to a particular bucket, along with a number of required optimizations to these processes to enhance the applications performance in various scenarios.

# Technical Details

This specification goes into details with regards to the "back-end" implementation of bootstrapping for an SDK, but does not define the high-level implementation which is visible to the user.  The SDK is responsible for implementing the following behaviours under the hood, the foundational document covers user-visible implementation.

### Memcached Connection Sequence

Individual memcached connections are described as being 'connected' only once the connection has been completely initialized at a memcached level (HELLO, ERRMAP, AUTH).  A connection can be either a cluster connection or a bucket connection, the differentiation solely being whether a SELECT_BUCKET has yet been performed on the connection.  Note that fetching a configuration from a connection is NOT considered part of the initialization of a memcached connection and is explicitly discussed outside the scope of the memcached initialization, though in the case of the pipelining that is performed, a valid operation may be sent as part of the initial connection pipeline sequence in order to reduce new-connection operational round trip time (RTT) to 1.

All memcached connections should first establish an appropriate network connection to the target node via TCP.  Once the connection is established, the client must perform a sequence of commands to initialize the connection.  This sequence must be performed in the order given in order to allow correct behaviour.  This sequence MUST be dispatched in a single batch (except in the case of SASL_CONTINUE) without waiting for responses.  The entire batch of initialization commands will be handled in sequence upon receiving responses, as described further below in this section.

- **HELLO**
  A HELLO operation must be dispatched to the node with a list of requested features along with a client identifier and connection UUID as per [SDK-RFC#35 Client Connection IDs][sdk-rfc-0035] and [SDK-RFC#47 Unified User Agent][sdk-rfc-0047].

- **GET_ERROR_MAP**
  A GET_ERROR_MAP operation must be sent.  This error map must be used by the client to decode errors received from this particular connection. (note that error maps do not cross between connections, and each connection must fetch their own error map).

- **SASL_LIST_MECHS**
  A SASL_LIST_MECHS operation must be dispatched to the server to identify a list of supported mechanisms.  The initial SASL_AUTH which is performed next will not be based on the results of this operation, instead it will be fetched pre-emptively in case the clients initial assumption about supported SASL mechanisms is incorrect.

- **SASL_AUTH**
  A SASL_AUTH operation must be dispatched to the server to authenticate the user.  The strongest supported mechanism supported by the client and enabled by the user should be used to transmit the users password (see the Authentication section for more details).  Later attempts to authenticate the user in the case of an unsupported authentication mechanism should utilize the mechanism list returned by the server at that point.

- **SASL_CONTINUE**
  In the case of SASL mechanisms which require continuation, the SDK will be required to wait for the SASL_AUTH response at this point, and then perform any necessary SASL_CONTINUE operations before proceeding.

- **SELECT_BUCKET**
  In the case that this connection is being used at a bucket level, and a bucket name is already available, the client must dispatch a SELECT_BUCKET operation to select the appropriate bucket.  This must occur prior to the cluster configuration request to ensure that a configuration containing bucket-specific context is returned.

- **GET_CLUSTER_CONFIG**
  A GET_CLUSTER_CONFIG operation must be dispatched in order to validate that the cluster that has been connected to is the one that is expected, and to confirm feature availability for this connection.

- **{A SINGLE OPERATION}**
  In order to enable the client to perform operations immediately with 1x RTT (even before initialization has occurred), the client MAY also dispatch any other pending idempotent operations following the previous commands.

Once the above batch of commands has been dispatched to the server to initialize the connection, the client must wait for the responses to the above batch of operations, and then depending on the result value of each individual operation, the client may proceed or decide to ignore the remaining results and re-attempt a particular segment of the initialization batch.  Note that any operations appearing later in the response stream MUST be ignored if an earlier step failed.

- **HELLO**

  - On success, record the list of supported features which the server has responded with and proceed with further initialization response processing.
  - On any error, report an error to the user indicating that the server version is too old to proceed with the connection.

- **GET_ERROR_MAP**

  - On success, update the connections internal error mapping and proceed with further initialization response processing.
  - On any error, the error map can be assumed to be blank, and all existing client defaults apply (SDK default error handling is discussed in [SDK-RFC#13 KV Error Map][sdk-rfc-0013]).  Proceed with further initialization response processing.

- **SASL_LIST_MECHS**

  - On success, record the list of supported mechanisms for this particular connection for future reference in case re-authentication is required in the response handling for SASL_AUTH.
  - On error, record that the SASL_LIST_MECHs returned an invalid response and proceed with the connection hoping that we don't need to list the mechanisms.  If a subsequent SASL_AUTH fails and we need the mechanism list, we will be required to cancel this connection and retry.

- **SASL_AUTH**

  - On success, proceed with further initialization response processing.
  - On invalid mechanism error, inspect the list of mechanisms reported by the server in the previous step and select a new best mechanism by matching the users configured mechanisms with the server supported mechanisms.  Initialization should be retried on the same tcp connection starting at the SASL_AUTH step.  Any further responses dispatched as part of the initialization batch MUST be ignored.  When a valid mechanism is found, this should be recorded by the SDK and used for all future connection attempts to the cluster.
  - On authentication error, the credentials should be marked as invalid and this should be bubbled upwards in the stack to the appropriate cluster or bucket connection logic as described in later sections of this document.  Note that the connections MUST be destroyed at this point, and any further retries must be performed against a new set of connections.
  - On other errors, cancel the connection attempt and retry.

- **SASL_CONTINUE**

  - Behaviour of SASL_CONTINUE response handling should mirror that of the SASL_AUTH operation above.

- **...**

- **GET_CLUSTER_CONFIG**

  - On success, proceed
  - On error when no bucket has been selected, mark the connection as not support G3CP and proceed.  The user will be required to open a bucket connection prior to executing any cluster-level operations to enable the cluster topology to be determined.
  - On error when a bucket has been selected, mark the connection as not supporting CCCP and proceed.  This can occur due to an older server version, or due the client connecting to a memcached bucket which does not support CCCP.  The client should subsequently spin up an HTTP watcher/poller to monitor for configuration changes through HTTP rather than CCCP.

- **...**
  - The results of any other operations which were performed as part of the initialization batch should be resolved directly back to the component that sent them (except in the case of any of the previously described failure scenarios, in which case the response should instead be ignored, and the operation dispatched again as part of the initialization retry batch).

### Cluster Connection (Bootstrapping) Sequence

Upon receiving a request from an application to establish a cluster connection, the client must parse the bootstrap list provided by the user according to [SDK-RFC#11 Connection Strings][sdk-rfc-0011].  This will produce a list of possible nodes to contact as a pair of hostname/port lists (the first being the memcached hostname/port's, the second being the http hostname/port's).  Once a list of hosts to contact has been established, the client must iterate through the list of memcached addresses from beginning until end, attempting to connect and fetch a configuration from each node until the entire list is iterated, or a node which explicitly does not support cluster-level configurations is encountered.  The SDK MUST perform these connections sequentially and not in parallel, and SHOULD perform this in a consistently random order (the SDK should pre-shuffle the list, and then use the pre-shuffled order consistently).  Depending on the results of this initial connection attempt, the client behaviour should branch based on the following:

- At least one node successfully connected and retrieved a cluster level configuration:

  - See the behaviour described in the remainder of this section.

- At least one node successfully connected but did not support GCCCP.

  - The connections which were previously established should be held 'in trust' awaiting a bucket connection which can consume them (as described in the section named 'Bucket Connection Sequence' in this document below).

- No node could be successfully contacted.
  - The client should proceed to attempt to connect again.  These reconnection attempts should continue until some form of connection is successfully established with the cluster.

Once a cluster-level configuration has been successfully retrieved from the cluster, the configuration should be stored to allow the client to perform cluster-level queries and management operations.  The configuration should also be used to establish connections to the remaining nodes from the cluster (such that there is exactly 1 established connection to each node in the cluster).

In addition to establishing further connections to the cluster, the client should begin refreshing the configuration periodically (typically 2.5s) from the cluster in round-robin fashion using any connected memcached connections as per [SDK-RFC#24 Fast Failover][sdk-rfc-0024] until the cluster is closed or a bucket is opened (cluster configuration switches to being derived from bucket configurations at that point, as described in the section named 'Bucket Connection Sequence' in this document).

### HTTP Fallback

During the CCCP phase of connecting, it is possible for the client to fall back to HTTP bootstrapping if the server or bucket type does not support CCCP.  In this case, the client should open a streaming bucket configuration connection to the server at `/pools/default/bs/$BUCKET_NAME`.  This will provide the client with a normal bucket configuration that can be used to perform bucket operations as well as infer the cluster configuration, as is done with a CCCP configuration.  In the case of a memcached bucket, the CCCP operation will fail and an HTTP fallback is expected to occur.  The client must destroy and refresh the HTTP streaming connection periodically in accordance with the configuration option, this is to ensure that dead connections are identified as soon as reasonable.

### Bucket Connection Sequence

Upon receiving a request from the application to open a bucket for operations, the client must begin the process of opening a bucket connection.

If the cluster object already has a set of connections which are established, the client MUST hijack these connections and perform a SELECT_BUCKET operation against them to upgrade the memcached connections to being bucket connections (rather than cluster connections).  Following this hijacking, any periodic configuration fetches being performed at the cluster level must be stopped (since the cluster object no longer owns any connections to perform those requests against).

If the cluster does not own any connections (too early, or another connected bucket already stole them), a new set of connections should be established instead.  This should follow the same semantics that are used for a cluster connection with the addition that should all connections to the cluster fail using memcached, the client should revert back to HTTP streaming, as described below.

Once connections are established for the bucket connection, the client should begin refreshing the configuration periodically under the same semantics used above for cluster level connections.

### Configuration Parsing

Below is an example of configurations as provided from the various different configuration sources.  Parsing of the configurations is relatively straightforward with each field reflecting its purpose.  The only caveat to this is in the case of KV nodes, the nodes list must be used to determine the bounds of nodesExt.  In some cases, the nodesExt list may contain more entries than the nodes list, in this case only the index-matched (against nodes) nodesExt entries should be considered for KV.

<https://gist.github.com/brett19/034c54d35959666868e35c4fac59277d>

### Authenticating

#### Authenticators

The SDKs expose authentication mechanisms through a concept familiar to SDK 2 named Authenticators.  Authenticators expose a (private) interface which enables the SDK to implement various methods of authenticating which are handled centrally by the Authenticator.  The Authenticator is also responsible for specifying the specific configurations they support (TLS vs Non-TLS for instance).  Note that while the interface for an authenticator is meant to be private, it is frequently used within couchbase to implement custom authentication, such as the one used for internal services (cbauth).

An example interface for your Authenticator might be:

```typescript
interface Authenticator {
  bool supportsTls()
  bool supportsNonTls()

  PrivateKey getCertificate(ServiceType, Endpoint)

  bool authKv(KvConnection)\
  bool authHttp(ServiceType, HttpRequest)
}
```

The SDK is expected to provide three default Authenticators, providing the ability to authenticate using Role-Based Access Control via a username and password, the ability to authenticate using a client certificate, and the ability to delegate to a different Authnenticator.

The RBAC Authenticator is expected to take a username and password as input from the user, and then use this information to perform SASL authentication on any KV connections, and to inject the HTTP Authorization header into HTTP requests. It does not provide any certificates for TLS connecting.

The Certificate Authenticator is expected to take a PrivateKey (or possibly simply a key name) from the user and use this information to provide a client-certificate for KV and HTTP connections alike.  It is also responsible for disabling the use of SASL_AUTH on connections as the server will already have authenticated the connection once its established using the provided client-certificate.

The DelegatingAuthenticator takes another Authenticator as a delegate, and forwards all operations to the delegate.  It has a public `setDelegate` method that allows the user to switch to a different delegate.  This lets the user refresh the Couchbase credential without having to restart the app.  SDK implementers must take care to ensure it's safe to call `setDelegate` at any time, from any thread, and concurrently with an authentication operation.  For example, in Java this requires marking the `delegate` field as `volatile` so the change is immediately visible to all threads.

An example implementation for the above mentioned authenticators might be:

```typescript
class CertificateAuthenticator {
  PrivateKey Key;

  supportsTls() { return true }
  supportsNonTls() { return false }

  getCertificate(ServiceType, Endpoint) { return this.Key }

  authKv(KvConnection) { return true }
  authHttp(ServiceType, HttpRequest) { return true }
}

class PasswordAuthenticator {
  String Username;
  String Password;

  supportsTls() { return true }
  supportsNonTls() { return true }

  getCertificate(ServiceType, Endpoint) { return nil }

  authKv(conn: KvConnection) {
    return conn.SASL_AUTH(this.Username, this.Password)
  }
  authHttp(type: ServiceType, req: HttpRequest) {
    req.headers['Authorization'] = this.Username + ':' + this.Password
    return true
  }
}

class DelegatingAuthenticator {
  Authenticator Delegate;

  setDelegate(delegate: Authenticator) { this.Delegate = delegate }

  // Also has all the usual methods, each one delegating
  // to the corresponding method of `Delegate`.
}
```

# Changelog

- Apr 1, 2019

  - Initial Draft

- Nov 19, 2019 - Revision #2 (by Brett Lawson)

  - Added section describing the implementation and behaviour of authenticators.
  - Removed section on individual bucket close as we only support closing whole cluster objects, not individual buckets.
  - Revised wording on KV bootstrap based on feedback (no actual behavioral changes were made).
  - Added explicit wording regarding node bootstrapping order being both consistently random and that it must be performed serially.

- Dec 19, 2019

  - Moved RFC to REVIEW state.

- Feb 11, 2019 - Revision #3 (by Brett Lawson)

  - Added paragraph explaining that this RFC covers the underlying behaviour of an SDK as opposed to the user-visible interface.

- April 30, 2020 (by Brett Lawson)

  - Moved RFC to ACCEPTED state.

- Sept 17, 2021 (by Brett Lawson)

  - Converted to Markdown

- Aug 22, 2025 - Revision #4 (by David Nault)

  - Added decription of DelegatingAuthenticator

# Signoff

| Language   | Team Member         | Signoff Date | Revision |
| ---------- | ------------------- | ------------ | -------- |
| Node.js    | Brett Lawson        | 2020-04-16   | 3        |
| Go         | Charles Dixon       | 2020-04-22   | 3        |
| Connectors | David Nault         | 2020-04-29   | 3        |
| PHP        | Sergey Avseyev      | 2020-04-22   | 3        |
| Python     | Ellis Breen         | 2020-04-29   | 3        |
| Scala      | Graham Pople        | 2020-04-30   | 3        |
| .NET       | Jeffry Morris       | 2020-04-22   | 3        |
| Java       | Michael Nitschinger | 2020-04-16   | 3        |
| C          | Sergey Avseyev      | 2020-04-22   | 3        |
| Ruby       | Sergey Avseyev      | 2020-04-22   | 3        |

[sdk-rfc-0007]: /rfc/0007-cluster_level_auth.md
[sdk-rfc-0011]: /rfc/0011-connection-string.md
[sdk-rfc-0016]: /rfc/0016-rbac.md
[sdk-rfc-0024]: /rfc/0024-fast-failover.md
[sdk-rfc-0035]: /rfc/0035-rto.md
[sdk-rfc-0047]: https://docs.google.com/document/d/1B4QM9UO6kz2yjLrBqLjSgArUeM1DvzKnakC_e8KfrmY/edit
