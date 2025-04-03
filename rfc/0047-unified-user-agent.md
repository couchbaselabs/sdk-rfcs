# Meta

* RFC Name: unified-user-agent
* RFC ID: 47
* Start Date: 2019-02-11
* Owner: Sergey Avseyev
* Contributors: Michael Nitschinger, Sergey Avseyev
* Current Status: Draft

# Summary

This RFC proposes the unification across SDKs of the “User Agent” identifier
which they send to the server on numerous occasions.

# Motivation

At the time of writing there are two ways the SDKs send identifying information
to the server: when sending a request via HTTP to services and during KV
"Hello" negotiation. While both are slightly different on the wire, they serve
a common purpose:

Identifying and "fingerprinting" the client to the server to aid subsequent
debugging and after-the-fact information gathering (i.e. "cbcollectinfo").

As a result, the more unified the user agent is across SDKs, the easier it is
for the field and automated tooling to extract and parse this information.

# General Design

There is already a very prominent precedent for user agents: The HTTP
User-Agent header (defined here
[https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/User-Agent](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/User-Agent)).

The SDKs send an actual HTTP User-Agent header to services like query or
analytics, but they also need to send a trimmed-down version via a KeyValue
HELLO negotiation. As a result, this RFC defines a more terse version of the
common HTTP equivalent:

    <sdk-identifier>/<version> [(<system-information>)]

The following sections break down each individual piece.

## SDK-Identifier

* Required

The identifier allows to uniquely distinguish between the different SDKs used.
The following identifiers must be followed by the implementations:

| SDK           | Identifier |
| ------------- | ---------- |
| .NET          | `dotnet`   |
| C++           | `cxx`      |
| Go            | `gocb`     |
| Go Core       | `gocbcore` |
| JVM Core      | `jvm-core` |
| Java          | `java`     |
| Kotlin        | `kotlin`   |
| Libcouchbase  | `lcb`      |
| Node.js       | `nodejs`   |
| PHP           | `php`      |
| Python        | `python`   |
| Ruby          | `ruby`     |
| Rust          | `rust`     |
| Scala         | `scala`    |
| Kafka         | `kafka`    |
| ElasticSearch | `es`       |
| Java Columnar      | `couchbase-java-columnar` |
| Node.js Columnar   | `nodejs-columnar`         |
| Python Columnar    | `python-columnar`         |
| Go Columnar        | `gocb-columnar`           |

Future identifiers should be added as amendments to this RFC as a single point
of reference.

## Version

* Required

The version must include the significant digits of the current followed semver
standard and can optionally include a platform-dependent suffix.

    major.minor.patch[suffix]

If for whatever reason a version cannot be determined, 0.0.0 must be used.

## System-Information

* Optional

The system information is the most flexible piece since it will inevitably be
different for each platform. Nonetheless a structure like the following must be
followed to help with automated parsing:

    ([os]; [platform])

Note that if neither the OS nor the platform are available, the surrounding
braces must be omitted completely. If only one of the two is available, the `;`
separator must be omitted.

* OS: The OS identifier, maybe including the architecture.
  For example: `Mac OS X/10.13.4 x86_64`
* Platform: If the SDK is running in a platform/runtime, include information.
  This could also include application server information.
  For example: `Java HotSpot(TM) 64-Bit Server VM 1.8.0_101-b13)`

# Short and Long Versions

The long version of the user agent is the version outlined above, not cut down
or shortened in any way. The long version is sent via HTTP to the following
services:

* Cluster Manager
* Views
* Query
* Analytics
* Search

The short version is only sent through the KeyValue service during the HELLO
command as part of the Client/Connection ID. Since the maximum string length of
the short version (as defined in [RFC\#35 Response Time
Observability](rfc/0035-rto.md#client--connection-ids)) is **200 characters**,
the long version must be trimmed if it is longer.

To avoid trimming vital information like the identifier and the version, it is
recommended that the excess characters should be removed from the system
informations if present. Note that 200 chars should hold plenty of room, this
example string is just 95 chars long:

    java/3.0.0 (Mac OS X/10.13.4 x86_64; Java HotSpot(TM) 64-Bit Server VM 1.8.0_101-b13)

# Language Examples

The following table shows the state of the user agent strings in different SDKs
as it implemented in most recent versions for November 6, 2024.

| SDK           | User Agent |
| ------------- | ---------- |
| .NET          | `couchbase-net-sdk/3.4.8.0 (clr/.NET 8.0.10) (os/Darwin 23.4.0 Darwin Kernel Version 23.4.0: Fri Mar 15 00:12:49 PDT 2024; root:xnu-10063.101.17~1/RELEASE_ARM64_T6020)` |
| Java          | `java/3.7.4 (Mac OS X 14.4.1 aarch64; OpenJDK 64-Bit Server VM 23.0.1)`   |
| Kotlin        | `kotlin/1.4.0 (Mac OS X 14.4.1 aarch64; OpenJDK 64-Bit Server VM 23.0.1)` |
| Libcouchbase  | `libcouchbase/3.3.13 (Darwin-23.4.0; arm64; AppleClang 15.0.0.15000309)`  |
| Scala         | `scala/1.7.4 (Mac OS X 14.4.1 aarch64; OpenJDK 64-Bit Server VM 23.0.1)`  |
| C++           | `cxx/1.0.6 (Darwin/arm64;bssl/0x1010107f;cbc)`                                                               |
| Ruby          | `ruby/3.5.6 (cxx/1.0.6;Darwin/arm64;bssl/0x1010107f;ruby_sdk/9bca6059;ruby_abi/3.4.0)`                       |
| Python        | `python/4.3.5 (cxx/1.0.5;Darwin/arm64;bssl/0x1010107f;python/3.9.6)`                                         |
| PHP           | `php/4.2.7 (cxx/1.0.6;Darwin/arm64;bssl/0x1010107f;php_sdk/4.2.7/d1ff155b;php/8.4.5/n)`                      |
| Node.js       | `nodejs/4.4.5 (cxx/1.0.5;Darwin/arm64;bssl/0x1010107f;node/21.2.0; v8/11.8.172.17-node.15; ssl/3.0.12+quic)` |

The following SDK at the moment do not follow the spec.

| SDK           | User Agent                      |
| ------------- | ------------------------------- |
| Go            | `gocbcore/v10.5.4 gocb/v2.9.4`  |

# Documentation

Since this is an internal behavior only, this RFC does not need to be
documented in the official user-facing documentation. Internal components
should refer to this RFC as a reference documentation.

# Changelog

* February, 2019 - Initial version
  * Removed prefix `cb-` and SDKs do not really implement it.
  * Added section describing behaviour of wrappers.
  * Updated list of the identifiers
  * Updated list of the examples showing current state of the things.

* February, 2019 - Revision #1
  See details and discussion at [original Google Drive document](https://docs.google.com/document/d/1B4QM9UO6kz2yjLrBqLjSgArUeM1DvzKnakC_e8KfrmY/edit).

# Signoff

| Language   | Team Member            | Signoff Date   | Revision |
|------------|------------------------|----------------|----------|
| .NET       |                        |                |          |
| Go         |                        |                |          |
| Java       |                        |                |          |
| Kotlin     |                        |                |          |
| Node.js    |  Jared Casey                      |   2025-04-03             |  1        |
| Python     | Jared Casey                       | 2025-04-03               | 1         |
| Scala      |                        |                |          |
| C          | Sergey Avseyev         | 2025-04-03     | 1        |
| C++        | Sergey Avseyev         | 2025-04-03     | 1        |
| PHP        | Sergey Avseyev         | 2025-04-03     | 1        |
| Ruby       | Sergey Avseyev         | 2025-04-03     | 1        |

