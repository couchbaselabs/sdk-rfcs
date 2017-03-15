# The SDK RFCs

> _Begin with the end in mind._
>
> Steven Covey

The SDK RFCs are where we think through the mechanics and semantics of new API being added to Couchbase.  The goal is to iterate on what the level of detail should be before writing a full implementation and to quickly surface things we may want to take into consideration in the design.  As it grows from an idea to shipping in a supported release, an SDK RFC will traverse along:

1. IDENTIFIED:  A need has been identified for an SDK RFC, but there's little more than a filed issue to point to at the moment
2. DRAFT: The owner of the SDK RFC has started to draft up how the subject will be handled and may be reviewing with a core group.  Comments are certainly welcome at this stage even though the owner hasn't worked through enough details to ask for...
3. REVIEW: This SDK RFC is in a review period. Stakeholders and the owner may still be iterating on some final details before signoff. A minimum review period has been defined.
4. ACCEPTED: All stakeholders have signed off and this SDK RFC is now or will be implemented soon.

Coding happens all the time and is encouraged. We just recognize there is a point where we will want to move from code having been written as a subproject or an experimental feature to a feature of the system.  At that point we want the feature to take into consideration more use cases and development platforms than it had when it was merely experimental.  We want the end result to feel like it's been designed, rather than merely written.

## SDK RFC Index

RFC #  | Description | Owner | Status
------------- | ------------- | --------- | ---------
1  | [The RFC Process](rfc/0001-rfc-process.md) | Matt | ACCEPTED
2  | [SubDocument API](https://docs.google.com/document/d/1ZXq-JgWW8ywU03Tx51A3jFTysYmlkv2W4ko0kepb3_M/edit#) | Mark | ACCEPTED
3  | [Index Management](https://github.com/couchbaselabs/sdk-rfcs/blob/master/rfc/0003-indexmanagement.md) | Simon | ACCEPTED
4 | [RYW Consistent Queries – at_plus](rfc/0004-at_plus.md) | Michael | ACCEPTED
5 | [VBucket Retry Logic](https://github.com/couchbaselabs/sdk-rfcs/blob/master/rfc/0005-vbucket-retries.md) | Brett | ACCEPTED
7 | Cluster Level Authentication [\[issue\]](https://github.com/couchbaselabs/sdk-rfcs/issues/13) [\[draft\]](https://docs.google.com/document/d/1CD5OL1ez7euCiLJT91zdWY9R4tW_bkGZ0wsFk1UDtyY/edit) | Brett | REVIEW
8 | [Datastructures](rfc/0008-datastructures.md) | Mark | ACCEPTED
9 | 2i Query Support | SDK | IDENTIFIED
10 | [Full Text Search (FTS) API](rfc/0010-cbft.md) | Michael | ACCEPTED
11 | [Connection String](rfc/0011-connection-string.md) | SDK | ACCEPTED
12 | ~~Adapt memcached error code handling for future proofing (superseded by RFC 13)~~ | SDK | SUPERSEDED
13 | [KV Error Map](https://docs.google.com/document/d/1OaOeQ2ex5anB2uNhQcOFMuYN2sbuvOSt20Cl7lTnj64/edit#heading=h.wmtpv6z0vz4w) | Mark | DRAFT
14 | ~~LWW Wins XDCR Support (superseded by RFC 17)~~ | SDK | SUPERSEDED
15 | Collection Support | SDK | IDENTIFIED
16 | [RBAC Support](https://docs.google.com/document/d/1SUFKCRIX8jp8q-NZy9ZpP4vs0pjLHkUlJFvZjcKr-DU/edit?usp=sharing) [\[issue\]](https://github.com/couchbaselabs/sdk-rfcs/issues/46) | Mike G | DRAFT
17 | Cross-Cluster Failover | Michael | IDENTIFIED
18 | Timeouts for Configuration and Operations | Michael | IDENTIFIED
19 | [SDK 2.0 APIs](https://docs.google.com/document/d/1HgVEJetcIfeIqviKC9zdlv_7IEkWpstatzxeydkLF3A) | Brett | ACCEPTED
20 | [Common Flags](https://docs.google.com/document/d/1V653a6FF6DOqdT4d-fKIjGkHabDaNGZsvbtsUKJyeLc) | Brett | ACCEPTED
21 | Generic find Queries ([draft discussion](https://github.com/couchbaselabs/sdk-rfcs/pull/54)) | Brett | DRAFT
22 | [User Management API](https://docs.google.com/document/d/1azNxFNLmcTvmhGc9-cAdswFgCyQR_s_lcfKCBE54gWE/edit?usp=sharing) | Subhashni | DRAFT
23 | [Subdoc `GET_COUNT` and `MKDOC`](https://docs.google.com/document/d/1RDhEs651weAoXVFuDkKRjZwxY9Q4rXhtA1zE49gRKQs) | Mark | DRAFT
24 | Fast failover configuration and behavior | Jeff | IDENTIFIED
25 | [XATTR Support](https://docs.google.com/document/d/1z3pJCPg77PZ8U8rFAyABuEhHSl68j1n42zP2RyflWZs/edit#heading=h.z5vaan7obomq) | Mark | DRAFT
26 | [Ketama Hashing](https://docs.google.com/document/d/129q1Eu6uKZD06UHMgermw1-A-oEsTIvA2a8_rdNKfew/edit?usp=sharing) [[issue]](https://github.com/couchbaselabs/sdk-rfcs/issues/56) | Mike G | REVIEW
27 | [CBAS Querying](https://docs.google.com/document/d/1EjdzVG4hVyunhoVVp7oAlPuNWvSAyqaHAMP0SJdUy0Q) | Brett L | DRAFT



[comment]: # (RFC States: IDENTIFIED > DRAFT > REVIEW > ACCEPTED)
[comment]: # (Description above must link to either the merged draft, the issue or the pull request when in any state otehr )


## Background

We take the addition or extension of API seriously because we intend this work to have a lifecycle of years or decades.  Toward that end, we had originally been writing a set of "one pagers" which had been influenced by experience in other software development organizations.  While those were great and we're even porting some of them over to RFCs, we recognized that we'd caught some mistakes late and want to focus ourselves on identifying affecting all platforms as early as possible.

Thus, we defined a new SDK RFC process.  We're not alone in this kind of endeavor.  Note that [Joyent have created RFDs](https://github.com/joyent/rfd) (which came from some similar experience [@ingenthr](http://github.com/ingenthr) and [@trondn](http://github.com/trondn) had at Sun as well), and [Rust has Rust RFCs](https://github.com/rust-lang/rfcs) in addition to the well known [IETF RFCs](http://ietf.org/rfc.html).

Even though this says "SDKs", frequently the discussion here will expose things not always considered at protocol and component design.  Those issues may be discussed in the SDK RFC or the discussion may spawn off to [Couchbase's issue tracker](https://issues.couchbase.com)
