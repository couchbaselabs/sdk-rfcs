# Meta
* RFC Name: Cluster Level Authentication
* RFC ID: 0007
* Start Date: 2016-01-12
* Owner: Brett Lawson (@brett19)
* Current Status: Accepted

# Summary
This RFC defines an implementation of cluster level authentication for the 2.x series of Couchbase Server client libraries.

# Motivation
In the current implementation of client libraries, authentication is provided on an as-needed basis. This means that a password is provided to the OpenBucket method, and this password is used throughout the clients to authenticate to various cluster services.  In addition, when a new manager object is created for a Cluster or Bucket, the credentials are provided at creation time.  The addition of cluster-level queries, which currently rely on the credentials of multiple buckets to execute, as well as the inclusion of more advanced form of cluster level authentication such as RBAC or CBAUTH have made it clear that an improved client authentication model is necessary.

# General Design
This specification proposes the addition of a high level and generic 'Authenticator' interface, which would be passed to a Cluster object by virtue of an 'Authenticate' method.  In addition to this generic interface, this specification will define the behaviour for an authenticator object which implements this interface according to the current authentication behaviour of Couchbase Server (PasswordAuthenticator).

The Authenticator interface is intended to be completely generic.  No expectations are set by this RFC in regards to the methods that it exposes or the internal implementation of this Authenticator.  The Authenticator MUST provide the ability for the client to fetch credentials for the following cases: bucket-level kv, bucket-level views, bucket-level management, cluster-level management, bucket-level n1ql, cluster-level n1ql, bucket-level cbft or cluster-level cbft.  These credential requests may require only a single username/password pair to be returned (such as for bucket kv), or may require a list of username/password pairs to be returned (such as for cluster-level N1QL queries).  The ability to fetch these tokens MUST NOT be exposed directly to the user (private methods). In addition, it should be noted that the Authenticator's internal behaviour is expected to change through sdk releases and MUST NOT be exposed publicly, as it may change even within patch versions.

The ClassicAuthenticator object is meant to expose our current paradigm of authentication behaviour.  It MUST store a list of bucket/password pairs, along with an optional cluster management username and password (Administrator/…).  When performing a bucket level authentication (kv, views, bucket-level n1ql query, etc…), the authenticator MUST return only the credential that matches the specified bucket from its list of bucket/password pairs.  When performing cluster-level management operations, the cluster management user/password MUST be returned.  When performing cluster-level operations, the entire list of bucket/password pairs MUST be returned.

Note that in all current cases where an SDK has provided the user with 'default credentials' (for instance, calling OpenBucket without a password), the SDK MUST now make an attempt to satisfy these credentials through the Authenticator interface.  For backwards compatibility, any SDK which currently has method overloads for passing passwords MUST use the password specified at the call-site over any passwords stored in the authenticator.  In this case, the SDK MUST NOT make changes to any user-defined Authenticator objects.

The following is a code example of a possible implementation:

```go
type Authenticator interface {
       // Note that the following is just an example, in this case, simply returning
       //  a username/password combination to use for the specific context.
	getCredentials(context string, specific string) (string,string, error)
}

Type ClassicAuthenticator struct {
	Buckets map[string]string
	Username string
	Password string
}

func (auth *ClassicAuthenticator) getCredentials(context string, specific string) ([][]string, error) {
	switch (context) {
	case "bucket-kv":
		return [][]string{ []string{ specific, auth.Buckets[specific] } }, nil
	case "bucket-n1ql":
		return [][]string{ []string{ specific, auth.Buckets[specific] } }, nil
	case "cluster-n1ql":
		return auth.Buckets, nil
	case "cluster-cbft":
		return auth.Buckets, nil
	case "cluster-mgmt":
		return [][]string{ []string{ auth.clusterUser, auth.clusterPass } }, nil
	}
	return nil, ErrAuthError
}

func (cluster *Cluster) Authenticate(authObj Authenticator) {
	cluster.auther = authObj
}
```

The following is a code example of a possible usage:

```go
auth := gocb.ClassicAuthenticator{
	Buckets: gocb.BucketAuthenticatorMap{
		"default": "default_password",
		"travel-sample": "go go gadet password"
	},
	Username: "Administrator",
	Password: "Charles Darwin",
}

cluster := gocb.Connect(...)
cluster.Authenticate(auth)

bucket := cluster.OpenBucket("default")
// Look ma', I have an authenticated bucket connection!
```

# Errors
No new errors will be introduced by the introduction of the authenticator objects, however it is reasonable to assume that the addition of future authenticators may surface new errors.

# Language Specifics
None known.

# Unresolved Questions
 - Perhaps the Authenticator class should instead be named SecurityContext?
   
   A: Authenticator will keep its name, the implementation however should be called 'ClassicAuthenticator' instead.
 - Use case: Make sure that there is a possibility to edit/change credentials in a running application.
   
   A: Due to the nature of the protocols we are dealing with, this RFC will not cover the case of attempting to allow the user to make 'live' modifications to the authenticator object.  Authenticator's on clusters/buckets shall be immutable (note that this enforces that changes to the authenticator do not change behaviour of objects already constructed using them, not that the object itself is immutable, ie: It is plausible that passing the authenticator to the Cluster would be by-value, implicitly making the clusters authenticator object immutable).
 - Java specific? The Java SDK already has an openBucket(String bucketName) method that default to the empty string for the password. So introduction of the Authenticator breaks the behavior. Should that method first look into the Authenticator for the password, and then default to empty string if not found?

   A: This is answered in the RFC above.
 - With regard to "edit/change" use case above, should this be made easier by exposing a getter for the Authenticator on the Cluster? And on the Bucket as well? Or should it be discouraged by exposing no such method in the public API interface? (maybe still expose it in the concrete classes, if relevant).

   A: See the above question regarding credentials on a running application.
 - So, we have an entry in the credentials dictionary for cluster-level N1QL creds; what if you pass credentials with a query, that doesn't require those credentials? I guess it's dependent upon the server impl., I am assuming it will only check that the credentials are valid for the cluster and not the specific query?

   A: When performing cluster-level queries, all available credentials at the cluster level of the Authenticator should be sent to the server (this means all bucket names/passwords).  It is not possible to determine the credentials that would be necessary to satisfy a particular query, so sending them all is the only implementable solution for the moment.  RBAC will mostly make this irrelevant anyways.
 - With RBAC coming, will the username not always be the bucket name in the future?

   A: In the future with RBAC, usernames will almost always be different from the buckets that the username has access to.  Note that with RBAC, a new authenticator object type will exist which only accepts a single username and password (the RBAC credentials).
 - Does "cluster-cbft" imply that the Cluster object will have a Query(statement or request) method in the future? If that is the case shouldn't Bucket.Query(statement or request) become obsolete and "bucket-n1ql" become obsolete as well? I know this is slightly out-of-scope of this RFC, but it begets the discussion. Same for CBFT - the API should be moved to the more global object (cluster).

   A: The Cluster object will definitely gain a Query method for performing cluster-level queries.  There was originally an RFC that existed for this, and an RI exists in both Node.js and Golang.  I can't seem to figure out what happened to that proposal/RFC though.

# Changelog
 - 05/18/2016: Changed name of first Authenticator implementation from 'ClusterAuthenticator' to 'PasswordAuthenticator'.
Reworded entirety of the RFC to better explain the motivation and behavioural expectations of implementations.

# Signoff

<table>
  <tr>
    <td>Language</td>
    <td>Representative</td>
    <td>Date</td>
  </tr>
  <tr>
    <td>Java</td>
    <td>Michael N.</td>
    <td>08/22/2016</td>
  </tr>
  <tr>
    <td>C</td>
    <td>Mark N.</td>
    <td>08/10/2016</td>
  </tr>
  <tr>
    <td>Python</td>
    <td>Mark N.</td>
    <td>08/10/2016</td>
  </tr>
  <tr>
    <td>.NET</td>
    <td>Jeff M.</td>
    <td>8/19/2016</td>
  </tr>
  <tr>
    <td>node.js</td>
    <td>Brett L.</td>
    <td>06/15/2016</td>
  </tr>
  <tr>
    <td>PHP</td>
    <td>Sergey A.</td>
    <td>08/10/2016</td>
  </tr>
  <tr>
    <td>Go</td>
    <td>Brett L.</td>
    <td>06/15/2016</td>
  </tr>
</table>
