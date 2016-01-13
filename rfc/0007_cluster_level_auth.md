# Meta

 - RFC Name: Cluster Level Authentication
 - RFC ID: 0007_cluster_level_auth
 - Start Date: 2016-01-12
 - Owner: Brett Lawson (@brett19)
 - Current Status: Draft

# Summary
This RFC defines an implementation of cluster level authentication for the 2.x series of Couchbase Server client libraries.

# Motivation
In our current implementation of the client libraries, our authentication is currently provided on a per-bucket basis.  This means that a password is provided to the `openBucket` method.  We are now facing a number of server changes which will challenge this existing authentication implementation by adding features such as role-based authentication, which allows a single set of credentials to provide access to multiple buckets, as well as numerous new features which are no longer scoped to a single bucket such as Cross-Bucket N1QL queries, CBFT, BI, etc...

# General Design
Add `authenticate` method at the cluster level which allows you to pass a generic 'authenticator' object.  The implementation of authenticator objects allows us to implement our current paradigm of having passwords for multiple discreet buckets, while still leaving room to permit us to expose RBAC/LDAP/etc... authentications in the future.  The authenticator objects will be responsible (internally) for accepting a context and returning an applicable set of credentials to use.  Some potential contexts include: cluster-management, bucket-kv, bucket-views, bucket-management, bucket-n1ql, cluster-n1ql, cluster-cbft.

## Errors
No new errors will be introduced by the introduction of the authenticator objects, however it is reasonable to assume that the addition of future authenticators may surface new errors.

# Language Specifics
None known.

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