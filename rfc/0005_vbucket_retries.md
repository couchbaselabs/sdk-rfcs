# Meta

 - RFC Name: VBucket Retry Logic
 - RFC ID: 0005_vbucket_retries
 - Start Date: 2016-01-12
 - Owner: Brett Lawson (@brett19)
 - Current Status: Draft

# Summary
This RFC defines a consistent implementation of vbucket retry logic for all Couchbase Server SDKs.

# Motivation
In the existing suite of client libraries there is no consistent behaviour defined in terms of how to perform vbucket retries.  This has resulted  in a variety of methods being employed, each with its own benefits and drawbacks.  We now need to align all the SDKs to have similar behaviour to ensure that performance expectations do not vary based on SDK.

# General Design
All client libraries must implement NMV status code interception to perform automatic retry.  Retry logic should be trivial and only consider the current map and fast-forward map which is available within our bucket configuration.  All operations should be attempted based on the current map, if the operation fails with NMV for the current map, then we should retry that operation using either the fast-forward map, or again with the current map should a fast-forward map not exist, every X milliseconds (100ms default, tuneable).  Note that operations for any particular vbuckets must be first attempted on the current map prior to being attempted on the fast-forward map, additionally, once an operation is retried on the fast-forward map reverting to the current map should not occur unless a configuration change is detected which indicates that the current map is once again valid.

This will result in an operation following these steps:

1. Dispatch Operation using current map
2. If not NMV, DONE
3. If Fast-Forward map exists, go to Step 6
4. Sleep 100ms
5. Go to step 1
6. Dispatch operation using fast forward map
7. If not NMV, DONE
8. Sleep 100ms
9. Go to step 6

## Errors
No errors will be changed by this proposal.  NMV replies MUST be caught and handled within the SDKs (never leaked to users).

# Language Specifics
No language specific API's or implementation details are required.

# Unresolved Questions

# Signoff

| Language | Representative | Date       |
| -------- | -------------- | ---------- |
| C        | | |
| Go       | | |
| Java     | | |
| .NET     | | |
| NodeJS   | | |
| PHP      | | |
| Python   | | |
| Ruby     | | |