# Meta

 - RFC Name: VBucket Retry Logic
 - RFC ID: 0005-vbucket-retries
 - Start Date: 2016-01-12
 - Owner: Brett Lawson (@brett19)
 - Current Status: Accepted (2016-06-15)

# Summary
This RFC defines a consistent implementation of vbucket retry logic for all Couchbase Server SDKs.

# Motivation
In the existing suite of client libraries there is no consistent behaviour defined in terms of how to perform vbucket retries. This has resulted in a variety of methods being employed, each with its own benefits and drawbacks. We now need to align all the SDKs to have similar behaviour to ensure that performance expectations do not vary based on SDK.

# General Design
All client libraries must implement NMV status code interception to perform automatic retry. Retry logic should be trivial and only consider the current map and fast-forward map which is available within our bucket configuration. All operations should be attempted based on the current map, if the operation fails with NMV for the current map, then we should retry that operation using either the fast-forward map, or again with the current map should a fast-forward map not exist, every X milliseconds (100ms default, tuneable). Note that operations for any particular vbuckets must first be attempted on the current map, meaning the map that may have changed since the last try, prior to being attempted on the fast-forward map. Additionally, once an operation is retried on the fast-forward map reverting to the current map should not occur unless a configuration change is detected which indicates that the current map is once again valid.  When a new map is received, the operation should be dispatched as soon as reasonable as defined by the SDK, this must be under under the tuneable value (default: 100ms), however it could be scheduled immediately.

This will result in an operation following these steps:

1. Dispatch Operation using current map
1. If not NMV, DONE
1. If Fast-Forward map exists, go to Step 6
1. Sleep 100ms
1. Go to step 1
1. Dispatch operation using fast forward map
1. If not NMV, DONE
1. Sleep 100ms
1. If received new config revision, GO to Step 1
1. Go to step 6

### Why 100ms linear retry?
The reason 100ms was chosen was because, it seemed conservative enough that it would definitely throttle the client talking to the network, so we don’t end up in a pathological situation.  But it is aggressive enough that a normal application probably would not notice the latency from a retry.  The reason behind not making it exponential, because of the rate of new incoming requests, if it is on a per-request basis, incoming operations could cause a pathological state where incoming requests continue to build on the load already causing the server to fail requests.

## Errors
No errors will be changed by this proposal. NMV replies MUST be caught and handled within the SDKs (never leaked to users).

# Language Specifics

No language specific API's or implementation details are required.

# Unresolved Questions
None.

# Changelog
 - 2016-05-06: Clarified behaviour of operations following a new configuration change being received by the SDK. (@brett19)

# Signoff

| Language | Representative | Date       |
| -------- | -------------- | ---------- |
| C        | Mark N.| May 6th, 2016 |
| Go       | Brett L.|  May 6th, 2016 |
| Java     | Michael N.| May 17th, 2016 |
| .NET     | Jeff M.| May 6th, 2016|
| NodeJS   | Brett L.| May 6th, 2016 |
| Python   | Mark N.| May 6th, 2016 |
| PHP      | Sergey A. | May 19th, 2016 |