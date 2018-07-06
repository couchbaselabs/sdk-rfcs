# Meta

 - RFC Name: Client Certificate Authentication
 - RFC ID: 0036-client-cert-auth.md
 - Start Date: 2017-12-06
 - Owner: Michael Nitschinger
 - Current Status: REVIEW
 
# Summary
The X.509 standard defines the format of public key certificates. In the context of Couchbase and this RFC, X.509-based (client certificate) authentication allows a user to authenticate via a certificate rather than via username/bucketname and password.

This SDK-RFC outlines the changes needed for each SDK to support this type of authentication and describes the semantics and limitations behind the approach.

# Motivation
Increasing customer demand for certificate-based authentication has made supporting it more important. 

An X.509 certificate does more than just distribute the public key: it is signed by a trusted (internal or third-party) CA, and thereby verifies the identity of the server, assuring clients that their information is not being sent to a rogue server.

Therefore, scenarios potentially requiring the use of X.509 certificates include:
 - In production, where clients have to go through untrusted networks.
 - When transferring sensitive data on the wire between application and Couchbase Server.
 - When mandated by compliance regulations.

As a result, all SDKs implementing this SDK-RFC in a unified way help our users to write more secure applications in a straightforward manner.

# General Design
Since the server establishes trust with the client through the imported certificate hierarchy, naturally the server and client need to work together in order to achieve the desired outcome.

How to create and import a X.509 certificate is out of scope for this RFC but can be found in our online documentation: https://developer.couchbase.com/documentation/server/5.0/security/security-x509certsintro.html

How certificates are managed and imported in each language/environment can differ, but there are a few important pieces which all SDKs need to be consistent on (also a useful checkbox list when writing tests):

 - Configuration
   - The user needs to signal through the configuration params that X.509 authentication is enabled (via the authenticator).
   - If X.509 is enabled, naturally all other config switches for SSL/TLS enabling must be turned on by the user as well (for SDKs using the connection string, if not “couchbases://” is used for example).
   - The certificate needs to be provided by the user so that it can be used during authentication.
 - User-Visible Behavior
   - In this case, all calls to bucket open attempts with a password need to be rejected since they are now semantically incorrect.
 - Internal Behavior
   - The SDK must not establish a non-encrypted socket if X.509 is turned on.
   - Every time a new socket is created the certificate needs to be sent to the server for authentication purposes.
   - All other forms of authentication (both SASL and HTTP auth) must be omitted once X.509 is turned on.

The next sections cover each of those in greater detail.

## Configuration
At configuration time, a couple of settings must align to correctly configure certificate authentication.

 1. A new authenticator introduced explicitly enables certificate authentication:
    
    `CertAuthenticator implements Authenticator (no args)`

    When this authenticator is used for the “authenticate” method, it signals to the client to pick up any certificate/keystore for X509 auth. In the future, it might be possible to use username/password credentials for auth but also use a certificate for trust but not identification.

    Note: The CertAuthenticator should follow the already established authenticator format in each respective SDK for consistency.

 2. In addition to the authenticator being defined, the SDK also needs to be able to pick up the certificate. This should follow the already established rules for TLS in each SDK, for example:
    - Java uses a keystore
    - .NET has a certstore
    - Libcouchbase uses certpath=/path/to/cert

 3. If the platform itself has on-by-default support for required, online, certificate revocation checking, a config switch must be in place to allow for disabling it (i.e. the .NET platform).
 
**Important**: setting a CertAuthenticator and failing to provide the appropriate certificate should be a hard config error during bootstrap. If not done the server will raise an authentication error, but to help with ease of debugging those cases an explicit error is preferrable. Also, if possible, certificate validation should be performed (as in: check for a valid username) but it is also optional. (but not the other way round to be future-proof as mentioned above).

## User-Visible Behavior
In addition to the configuration changes, it is important to early on alert the user of incompatible API calls:

 - `openBucket(name, password)`, in most SDKs found at the Cluster level. Password buckets are not needed since authentication is performed through the certificate (similar error when RBAC auth is used and a password is set on openBucket).

## Internal Behavior
As with all the SSL settings in general, the SDK must not establish non-SSL sockets if the config switch is flipped on. In addition to this, the SDK needs to make sure to send the stored certificate to the server on SSL/TLS connection handshake, since that’s the only way the server will know which user is authenticating.

Also, there are service specific behavior changes needed:

 - when initializing the KV sockets, the SASL authentication negotiation must be omitted completely.
 - HTTP services must not send the basic auth header at all but rather rely on the previous TLS certificate negotiation phase.

## Server Behavior
Ns_server (and for our purposes we treat the individual services the same), has three modes of configuration for X.509: “disable”, “enable” and “mandatory”. 

 - *disable*: Disables client based certificate authentication. This is the default value.
 - *enable*: When enabled, if the client presents a certificate, then that certificate is used to authenticate. If authentication fails, then access is denied. However, if the client does not present a certificate, the certificate based authentication will be bypassed.
 - *mandatory*: The client must present a valid certificate in order to gain access to Couchbase buckets. If using XDCR, do not use the mandatory state for X.509 Certificate Authentication.

From an SDK perspective enable and mandatory are similar, but from a user perspective when set to “mandatory” it means that all other means are basically disabled, leading to authentication errors when connecting and not using the X.509 approach.

## Documentation
General documentation has been produced for the server which also contains instructions on how to set it up: https://developer.couchbase.com/documentation/server/current/security/security-x509certsintro.html

The SDK documentation needs to focus on the language-specific ways to enable this mode of authentication and discuss potential errors in edge cases and misconfigurations outlined.

## Errors & Error Handling
### Unsupportive Services
If a service does not support X.509 cert authentication, the following errors are returned to the SDK (if we send a X.509 certificate during the TLS handshake, then it doesn’t matter if a basic auth header is provided or not, same response on both):

- Config/ns_server: 401 Unauthorized
- Query: returns with 200 OK and part of the regular error payload is 
`{"msg":"User does not have credentials to run SELECT queries on the travel-sample bucket. Add role query_select on <USERNAME> to allow the query to run.","code":13014}`
Checking on the error code might be the simplest thing to do.
- FTS: returns with 403 Forbidden and non-json `rest_auth: cbauth.AuthWebCreds, err: Authentication failure.`
- Views: returns with 401 unauthorized and `{"error":"unauthorized","reason":"password required"}`.
Important: the current retry rules mandate that 401 is being retried, so this would end in a retry loop (until the timeout/max request liftetime hits) not checked against the config setting and being short-circuited.

In all three cases, the SDKs must check for that specific error and IF x509 is enabled through the configuration, raise a `CertAuthNotSupportedException` or a similar error code to make it clear to the user whats going on (since the individual errors are ambiguous).

### KV Engine And X.509
As soon as the certificate with credentials is passed to KV engine over 11210, subsequent SASL authentication mechanisms are not supported anymore. This means that for example a LIST_MECHS returns with a 0x83 NOT_SUPPORTED, since the auth has already been performed.

As a result, as soon as client certificate auth is enabled, all SASL steps need to be skipped and not performed as part of the bootstrap negotiations. Note that SELECT_BUCKET is not considered to be part of SASL auth and as a result still needs to be performed (otherwise since just a username is given, a bucket to use still needs to be negotiated).

### Client Bootstrap

- When everything is set up properly but then openBucket gets called with a password, a `AuthenticationException` or similar needs to be thrown indicating that this combination doesn’t work. The exception/error is similar to the one where RBAC authentication is used and a password is provided during open bucket.
- In SDKs where possible illegal combination of Authenticator and SSL credential input (key, keystore, truststore…) should be detected and raised. This is optional and not specified in the RFC because of platform specific details.

Specifically the following cases should be handled (and to be raised with a AuthenticationException compatible error from the language table below):

A CertAuthenticator is used AND bucket credentials are specified on openBucket().

- If the environment/connection string is configured to use certificate authentication but no CertAuthenticator OR a different Authenticator is specified
- If a `CertAuthenticator` is used but the environment/connection string is not configured with the proper configuration.

## Certificate and Credentials
It is also possible that a certificate is provided which does not contain credentials. The SDK needs to properly react to different circumstances as the following table shows:

|                   | Certificate Without Creds                                                                                | Certificate Without Creds                                                                          |
|-------------------|----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| certAuth Enabled  | - Expected - uses credentials from cert - Errors might still happen to unsupportive services (see above) | - SASL is skippedno http auth header sent,  - leading to auth errors - Doesn’t work                |
| certAuth Disabled | - SASL errors (see above) - HTTP creds ambiguous                                                         | - Expected - uses credentials from cluster.authenticate() or even from openBucket with old servers |
