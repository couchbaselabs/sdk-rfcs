# The SDK RFCs

> _Begin with the end in mind._
>
> Steven Covey

The SDK RFCs are where we think through the mechanics and semantics of new API being added to Couchbase.  The goal is to iterate on what the level of detail should be before writing a full implementation and to quickly surface things we may want to take into consideration in the design.  As it grows from an idea to shipping in a supported release, an SDK RFC will traverse along:

1. **Identified**:  A need has been identified for an SDK RFC, but there's little more than a filed issue to point to at the moment
2. **Draft**: The owner of the SDK RFC has started to draft up how the subject will be handled and may be reviewing with a core group.  Comments are certainly welcome at this stage even though the owner hasn't worked through enough details to ask for...
3. **Review**: This SDK RFC is in a review period. Stakeholders and the owner may still be iterating on some final details before signoff. A minimum review period has been defined.
4. **Accepted**: All stakeholders have signed off and this SDK RFC is now or will be implemented soon.

Coding happens all the time and is encouraged. We just recognize there is a point where we will want to move from code having been written as a subproject or an experimental feature to a feature of the system.  At that point we want the feature to take into consideration more use cases and development platforms than it had when it was merely experimental. We want the end result to feel like it's been designed, rather than merely written.

## SDK RFC Index

### Accepted RFCs

RFC #  | Description | Owner | Status
------------- | ------------- | --------- | ---------
1  | [The RFC Process](rfc/0001-rfc-process.md) | Matt | ACCEPTED
5 | [VBucket Retry Logic](https://github.com/couchbaselabs/sdk-rfcs/blob/master/rfc/0005-vbucket-retries.md) | Brett | ACCEPTED
11 | [Connection String](rfc/0011-connection-string.md) | SDK | ACCEPTED
13 | [KV Error Map](rfc/0013-kv-error-map.md) | Brett (Mark) | ACCEPTED
20 | [Common Flags](rfc/0020-common-flags.md) | Brett | ACCEPTED
24 | [Fast failover configuration and behavior](rfc/0024-fast-failover.md) | Jeff | ACCEPTED
26 | [Ketama Hashing](rfc/0026-ketama-hashing.md) | Mike G | ACCEPTED
30 | [Client-Side Compression](rfc/0030-compression.md) | Sergey | ACCEPTED
32 | [Field-Level Encryption](rfc/0032-field-level-encryption.md) | Jeff | ACCEPTED
48 | SDK3 Bootstrapping [\[gdoc\]](https://docs.google.com/document/d/1SUSBM9XoTnpaeew0bq4ABmgISc76fjzrY2vapZ10XD4/edit#) | Brett | ACCEPTED
49 | SDK3 Retry Handling [\[gdoc\]](https://docs.google.com/document/d/1h9YL2k8uzNPnxN_XanagEc2d8qRVu6sVDiKaqASqjS4/edit#heading=h.95ju11kiipzo) | Michael | ACCEPTED
50 | SDK3 Datastructures [\[gdoc\]](https://docs.google.com/document/d/1mKk20ScVE8ssF2DvqZTe9xIUvOUanJ7LOARFiCJPkQ0/edit) | Brett | ACCEPTED
51 | SDK3 Views [\[gdoc\]](https://docs.google.com/document/d/10urBYts5kHH7TGnIowoyISYdDie70ARpxbnajfY43C8/edit) | Sergey | ACCEPTED
52 | SDK3 Full Text Search [\[gdoc\]](https://docs.google.com/document/d/1El3Kll05woPLxH39YyFUg7lRwfaaini-QS78CpstTLY/edit)  | Sergey | ACCEPTED
53 | SDK3 CRUD API [\[gdoc\]](https://docs.google.com/document/d/1_fPJn9trqG6e7iTpzqwCnurvxmBlguFVjh00F2Co7Y8/edit) | Jeff | ACCEPTED
54 | SDK3 Management APIs [\[gdoc\]](https://docs.google.com/document/d/1p6AUV_YKJBqaybt_JFmDTYjjIZ3t4Lp2uqJSFY0VtjY/edit#) | Charles | ACCEPTED
55 | SDK3 Transcoders & Serializers[\[gdoc\]](https://docs.google.com/document/d/1Fm7LSly5eKGpi-tVwlLGpe-ii1AT0p39ICqXysOJq8E) | Jeff | ACCEPTED
56 | SDK3 Query API [\[gdoc\]](https://docs.google.com/document/d/1MigpPIoEc-srXRJb7bUel3qTlGPvI-be-hT0jkGNNAY/edit) | Michael | ACCEPTED
57 | SDK3 Analytics API [\[gdoc\]](https://docs.google.com/document/d/1mpDBR3_-jmMRelWkNUlGazgVWPhF_15vhP5m5mB3wo4/edit) | Michael | ACCEPTED
58 | SDK3 Error Handling [\[gdoc\]](https://docs.google.com/document/d/1oWyuPIxmErNh9BerU3UYqpFQPnmg9Yl8h7oNJzCVVYM/edit) | Jeff | ACCEPTED
59 | SDK3 Foundation [\[gdoc\]](https://docs.google.com/document/d/1pt8wrSu7xvaqjG5vxcQSZN1epw6oP4MyTcZZMvSYwQo/edit) | Brett | ACCEPTED
61 | SDK3 Diagnostics [\[gdoc\]](https://docs.google.com/document/d/1Lw3nuYVtRbXYIujeCxakcyYQWFs2OAj4Borb75eNe5Y/edit?usp=sharing) | Michael N. | ACCEPTED

### Draft & Review RFCs

RFC #  | Description | Owner | Status
------------- | ------------- | --------- | ---------
29 | Server Version Identification [\[gdoc\]](https://docs.google.com/document/d/1d6j0R0BFloQgoQ981PjAzv2AWfAIRPlkBLvlCMG7ipY/edit?usp=sharing) | Mike G | DRAFT
38 | Enhanced Prepared Statements [\[gdoc\]](https://docs.google.com/document/d/1JhprmvL2HwHzkg7GxouGJc67eAvKFJekgyOG23T8mVU/edit)| Michael | DRAFT
39 | Multi Network Configurations [\[gdoc\]](https://docs.google.com/document/d/1706x2zMsYoBXQ-8H0cpW0KDYpeBy_FZ9dt1--NnQIzk) | Brett | DRAFT
45 | Advanced Analytics Querying [\[gdoc\]](https://docs.google.com/document/d/1SRYPk4ATM2PVc2Yi3WP-Ol9_qvFue9IG2uhd0UUq9GY) | Michael | DRAFT
46 | Synchronous Replication [\[gdoc\]](https://docs.google.com/document/d/1_Bn_cKLxvqFBNVcPaPnoXMpt3JEbf_6MDvMHpJDtO_s/edit) | Sergey | DRAFT
47 | Unified User Agent [\[gdoc\]](https://docs.google.com/document/d/1B4QM9UO6kz2yjLrBqLjSgArUeM1DvzKnakC_e8KfrmY/edit?usp=sharing) | Michael | DRAFT
60 | SDK3 Response Time Observability [\[gdoc\]](https://docs.google.com/document/d/11s2QCIBB-koFUm0ZzWI6aBy27hdRoDc0cRHsWvDT-xI/edit) | Mike G | DRAFT
62 | Eventing Management APIs [\[gdoc\]](https://docs.google.com/document/d/1VSqyRjFHJvlr9kYlwzeUpSDC8QkeflTm1epH7UzL0yw) | Michael N. | DRAFT
63 | User Impersonation [\[gdoc\]](https://docs.google.com/document/d/18FTOTIHktHjrntMT2A4qApZco7i5FZwlTEqUcyaquqo/edit#) | Brett L. | DRAFT
64 | SDK3 Field-Level Encryption [\[gdoc\]](https://docs.google.com/document/d/1JNp6a0_xdYu2kuV2wIb8mlfD885l0G939CP-r6qBQhw/edit#) | David N. | DRAFT
65 | CreateAsDeleted Support [\[doc\]](https://docs.google.com/document/d/1QccFEvHWEL2-ldS_aTfjphYJGB4YVmMKrMB-UyL2KFI/edit?usp=sharing) | Graham P. | DRAFT

### Identified RFCs

RFC #  | Description | Owner | Status
------------- | ------------- | --------- | ---------
9 | 2i Query Support | SDK | IDENTIFIED
15 | Collection Support | Brett | IDENTIFIED
17 | Cross-Cluster Failover | Michael | IDENTIFIED
18 | Timeouts for Configuration and Operations | Michael | IDENTIFIED
21 | Generic find Queries [\[issue\]](https://github.com/couchbaselabs/sdk-rfcs/pull/54) | Brett | IDENTIFIED
40 | Feeds | Brett | IDENTIFIED
41 | Multi-Document Atomicity | Graham | IDENTIFIED
42 | Config Publish Interleave | Charlie | IDENTIFIED
44 | XDCR Durability | SDK | IDENTIFIED

[comment]: # Next RFC ID 65

### Superseded and Deprecated RFCs

RFC #  | Description | Owner | Status
------------- | ------------- | --------- | ---------
2  | ~~[SubDocument API](rfc/0002-subdocapi.md)~~ | Mark | SUPERSEDED
3  | ~~[Index Management](https://github.com/couchbaselabs/sdk-rfcs/blob/master/rfc/0003-indexmanagement.md)~~ | Simon | SUPERSEDED
4 | ~~[RYW Consistent Queries – at_plus](rfc/0004-at_plus.md)~~ | Michael | SUPERSEDED
7 | ~~[Cluster Level Authentication](rfc/0007-cluster_level_auth.md)~~ | Brett | SUPERSEDED
8 | ~~[Datastructures](rfc/0008-datastructures.md)~~ | Mark | SUPERSEDED
10 | ~~[Full Text Search (FTS) API](rfc/0010-cbft.md)~~ | Michael | SUPERSEDED
12 | ~~Adapt memcached error code handling for future proofing~~ (see RFC 13) | SDK | SUPERSEDED
14 | ~~LWW Wins XDCR Support~~ (see RFC 17) | SDK | SUPERSEDED
16 | ~~[RBAC](rfc/0016-rbac.md)~~ | Mike G | SUPERSEDED
19 | ~~[SDK 2.0 APIs](https://docs.google.com/document/d/1HgVEJetcIfeIqviKC9zdlv_7IEkWpstatzxeydkLF3A)~~ | Brett | SUPERSEDED
22 | ~~[User Management API](rfc/0022-usermgmt.md)~~ | Subhashni | SUPERSEDED
23 | ~~Subdoc `GET_COUNT` and `MKDOC`~~ (see RFC 25) | Mark | SUPERSEDED
25 | ~~[Spock additions for Subdoc (including XATTRs)](rfc/0025-subdoc-xattr.md)~~ | Brett (Mark) | SUPERSEDED
27 | ~~[Analytics Querying](rfc/0027-analytics.md)~~ | Michael (Brett) | SUPERSEDED
28 | ~~[Enhanced Error Messages](rfc/0028-enhanced_error_messages.md)~~ | Brett L | SUPERSEDED
31 | ~~Custom Transcoders [\[draft\]](https://docs.google.com/a/couchbase.com/document/d/1p3VzB41Tv-q0-j_HsqJAUrijAJEB9rGJ92Qgf36JdXc/edit)~~ | Mike G | SUPERSEDED
33 | ~~Circuit Breakers [\[draft\]](https://docs.google.com/document/d/1QVXMN2u9RUuOAEPbeRvEA8h6drDJKR9Jy1C1Op17q3U/edit#)~~ | Michael N | SUPERSEDED
34 | ~~[Health Check](rfc/0034-health-check.md)~~ | Sergey | SUPERSEDED
35 | ~~[Response Time Observability](rfc/0035-rto.md)~~ | Mike G | SUPERSEDED
36 | ~~[Client Certificate Authentication](rfc/0036-client-cert-auth.md)~~ | Michael | SUPERSEDED
37 | ~~FTS Index Management [\[draft\]](https://docs.google.com/document/d/1C4yfTj5u6ahRgk3ZIL_AkwPMeu9-hHY_lZcsDNeIP74/edit?usp=sharing)~~ | Subhashni | SUPERSEDED
43 | ~~Enhanced Durability Requirements~~ (see RFC 46)| Michael | SUPERSEDED


[comment]: # (RFC States: IDENTIFIED > DRAFT > REVIEW > ACCEPTED)


## Background

We take the addition or extension of API seriously because we intend this work to have a lifecycle of years or decades.  Toward that end, we had originally been writing a set of "one pagers" which had been influenced by experience in other software development organizations.  While those were great and we're even porting some of them over to RFCs, we recognized that we'd caught some mistakes late and want to focus ourselves on identifying affecting all platforms as early as possible.

Thus, we defined a new SDK RFC process.  We're not alone in this kind of endeavor.  Note that [Joyent have created RFDs](https://github.com/joyent/rfd) (which came from some similar experience [@ingenthr](http://github.com/ingenthr) and [@trondn](http://github.com/trondn) had at Sun as well), and [Rust has Rust RFCs](https://github.com/rust-lang/rfcs) in addition to the well known [IETF RFCs](http://ietf.org/rfc.html).

Even though this says "SDKs", frequently the discussion here will expose things not always considered at protocol and component design.  Those issues may be discussed in the SDK RFC or the discussion may spawn off to [Couchbase's issue tracker](https://issues.couchbase.com)
