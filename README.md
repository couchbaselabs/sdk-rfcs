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

RFC #  | Description | Status 
------------- | ------------- | --------- 
1  | [The RFC Process](https://github.com/couchbaselabs/sdk-rfcs/blob/master/rfc/0001-rfc-process.md) | ACCEPTED 
2  | [SubDocument API](https://github.com/couchbaselabs/sdk-rfcs/blob/master/rfc/0002-subdocapi.md) | DRAFT 
3  | [Index Management](https://github.com/couchbaselabs/sdk-rfcs/issues/6) | DRAFT 
4 | RYW Consistent Queries– at_plus | IDENTIFIED 
5 | [VBucket Retry Logic](https://github.com/couchbaselabs/sdk-rfcs/issues/10) | DRAFT 
7 | Cluster Level Authentication | IDENTIFIED 
8 | Datastructures | IDENTIFIED 
9 | 2i Query Support | IDENTIFIED 
10 | CB Full Text API | IDENTIFIED 

[comment]: # (RFC States: IDENTIFIED > DRAFT > REVIEW > ACCEPTED)
[comment]: # (Description above must link to either the merged draft, the issue or the pull request when in any state otehr )


## Background

We take the addition or extension of API seriously because we intend this work to have a lifecycle of years or decades.  Toward that end, we had originally been writing a set of "one pagers" which had been influenced by experience in other software development organizations.  While those were great and we're even porting some of them over to RFCs, we recognized that we'd caught some mistakes late and want to focus ourselves on identifying affecting all platforms as early as possible.

Thus, we defined a new SDK RFC process.  We're not alone in this kind of endeavor.  Note that [Joyent have created RFDs](https://github.com/joyent/rfd) (which came from some similar experience [@ingenthr](http://github.com/ingenthr) and [@trondn](http://github.com/trondn) had at Sun as well), and [Rust has Rust RFCs](https://github.com/rust-lang/rfcs) in addition to the well known [IETF RFCs](http://ietf.org/rfc.html).

Even though this says "SDKs", frequently the discussion here will expose things not always considered at protocol and component design.  Those issues may be discussed in the SDK RFC or the discussion may spawn off to [Couchbase's issue tracker](https://issues.couchbase.com)
