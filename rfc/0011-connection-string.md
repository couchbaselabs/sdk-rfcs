# Meta

 - RFC Name: Connection String
 - RFC ID: 0011
 - Start Date: 2014-02-12
 - Owner: Brett Lawson
 - Current Status: Accepted

# Summary
Define a URI-like connection string format for specifying bootstrapping and connection options for Couchbase.

# Motivation
Currently there is a number of variables that define the way that a cluster can be connected to.  This can be defined as a matrix with 3 axes: SSL/Plain, HTTP/CCCP, SRV/Direct.  Currently we are planning to provide a number of different specifications between client libraries to connect to a cluster, whereas we should be providing one unified specification which defines all neccessary details and semantics explicitly.  Additionally, this proposal will attempt to define the precise semantics for some edge scenarios which have not yet been defined.

# General Design
A connection string will be consist of a scheme followed by host list (which potentially includes a port) along with a a set of key-value pairs.  This should follow the existing format for a URI, except multiple hosts-port pairs will be supported by using a `,` separated.  Additionally, `;` must be supported as a valid seperator for backwards compatiblity.

Two base URI schemes must be supported by all clients, these are the `couchbase` scheme along with the `http` scheme for backwards compatability.  The `couchbase` scheme should use CCCP only to perform connections and is expected to be used with all new clusters.  The `http` scheme will behave as current clients do, performing neccessary heuristics to attempt CCCP connections prior to HTTP fallback (exact heuristics defined below).  If no scheme is provided, the `http` scheme should be assumed, including all related heuristics.

In addition to the two base URI schemes, there will be an additional supported scheme `couchbases`, this will utilize the exact bootstrap and connection semantics as the `couchbase`, but will utilize SSL encrypted channels only for cluster communication.  Note that the use of `couchbase` means the client must not use SSL, and `couchbases` means the client must use SSL.  Note that the https scheme is not valid.

Following the scheme, a list of hosts should be provided, with optional ports specified on a per-host basis.  This list should be delimited by a `,`, or alternatively a `;` for backwards compatability.  In addition, a single host with no optional port being specified should cause a SRV-record lookup to be performed prior to attempting A-record lookup.  See below for the layout of SRV-records.  Should no port be explicitly specified, the services default canonical port should be used.

### HTTP Scheme Heuristics
If a client specifies the `http` scheme, and does not specify a port or specifies the default canonical service port (8091), then an attempt should be made to utilize CCCP via 11210 prior to attempting to request the configuration via HTTP.  Should a non-default port be specified, a HTTP request should be made immediately without attemping CCCP first.  Additionally, when performing bootstrapping with this heuristic mode, all hosts should be attempted for CCCP prior to HTTP.

### SRV Records
A DNS SRV record lookup should be performed should the connection string match the following criteria:
Contains only a single hostname
(ie: `couchbase://bb.org` not `couchbase://bb.org,cc.org`).
Provides no explicit port specifier
(ie: `couchbase://example.org`, not `couchbase://example.org:11210`).
The scheme matches `couchbase` OR `couchbases`.
(ie: `couchbase://example.org` but not `http://example.org`)

If all of these conditions are met, a DNS lookup should be performed for `_[$scheme]._tcp.[$hostname]` (ie: a connection string of `couchbases://example.org/` should perform a SRV record lookup for `_couchbases._tcp.example.org`).
Should any SRV records exist for this hostname:
The host/port list within the connection string should be replaced with the target/port from all returned records.  The original hostname that was used to perform the lookup should not be included in the list, unless it also exists as an SRV record.
Should no SRV records be found for the hostname:
The connection string should be remain as-is, and no further SRV related processing need occur.


SRV records will be defined with no specific logic regarding weighting and priority at this point in time.  Documentations should reflect that these values MUST be 0, however clients MUST also ignore these values.

At the time of writing, two possible valid SRV prefixes are considered valid.  Clients should ensure to only query for the record matching the scheme of the URL, clients should ensure to only query the record matching the active security mode.
_couchbase._tcp.* (for non-secured memcached)
_couchbases._tcp.* (for SSL-secured memcached)


### IPv6 Addresses
IPv6 addresses will be specified similar to the IETF RFC-2732 document.  The literal address will be enclosed within "[" and "]" characters to disambiguate a IPv6 octal separator and an explicit port.  Note that IPv4 addresses written in IPv6 notation (i.e. ::192.9.5.5, ::ffff:192.0.2.128) should still be resolved as IPv4 addresses within a client.

### Additional Points
The user-specified bootstrap list should be considered invalid and replaced by the server configuration as soon as the first valid configuration is received from the server.
Username and passwords should be passed outside of the connection string, and not as part of it.
All key/value pairs should use lower-case, underscore seperated names.


### Examples
- Valid
  - 10.0.0.1:8091 (deprecation warning)
  - http://10.0.0.1
  - couchbase://10.0.0.1
  - couchbases://10.0.0.1:11222,10.0.0.2,10.0.0.3:11207
  - couchbase://10.0.0.1;10.0.0.2:11210;10.0.0.3
  - couchbase://[3ffe:2a00:100:7031::1]
  - couchbases://[::ffff.192.168.0.1]:11207,[::ffff.192.168.0.2]:11207
  - couchbase://test.local:11210?key=value
  - http://fqdn
  - http://fqdn?key=value
  - couchbases://fqdn
- Invalid
  - http://host1,http://host2
  - https://host2:8091,host3:8091
  - http://::ffff:00ee:2122

# Unresolved Questions
None

# Signoff
| Language | Representative     | Date        |
| -------- | ------------------ | ----------- |
| C        | Mark Nunberg       | ~2014-07-01 |
| Java     | Michael Nitchinger | ~2014-07-01 |
| .NET     | Jeff Morris        | ~2014-07-01 |
| NodeJS   | Brett Lawson       | ~2014-07-01 |
| PHP      | Brett Lawson       | ~2014-07-01 |
| Python   | Mark Nunberg       | ~2014-07-01 |
| Ruby     | Sergey Avseyev     | ~2014-07-01 |


# Change Log
 - May 23, 2014
   - Renamed project from 'Project Unify-My-Bootstrapping-Please'
   - Defined additional semantics for missing ports in the hosts list.
 - May 28, 2014
   - Update to using URI schemes.
 - June 20, 2014
   - Various minor changes related to open questions.


> This document is copy of the document available below and was converted to an sdk-rfc on Jan 18th, 2016.
> https://docs.google.com/document/d/172ausWsYt3eYYOZ1lYHVS8ccbrrVJaGwHIRsf_O_Hyc
