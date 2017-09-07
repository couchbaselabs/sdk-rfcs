## Meta

 * RFC Name: User Management Support
 * RFC ID: 22
 * Start Date: 2017-08-03
 * Owner: Subhashni Balakrishnan
 * Current Status: Review

## Summary
The goal of this RFC is to introduce SDK APIs to manage users for RBAC (Role-Based Access Control).
An user with appropriate administrative access is able to create/update/delete users and get all users info.

## Motivation
This ability allows for administrative tools development using the SDK and also for testing purposes.

## General Design
The user management API should be added to the cluster management interface existing in SDK and internally they will interact with `ns_server`. There are two user types in RBAC - `local` and `external`. Local users are managed by the server while the external (LDAP) users are role-mapped.

The user types are referred to as `AuthDomain`:

```
Enum AuthDomain {
  LOCAL, // on the wire its "local"
  EXTERNAL // on the wire its "external"
}
```

## API

### `upsertUser`
Creates/updates a Couchbase user.

```
boolean upsertUser(AuthDomain domain, String userid, UserSettings settings)
```

If the domain is `external` and a password is set in the user settings, the client should proceed with sending the request to the server and log a warning saying that the password cannot be updated for external users.

`UserSettings` are described with the following properties:

```
UserSettings {
	String password;
	String name;
	Role[] roles;
}

Role {
	String role;
	String bucket_name;
}
```

**Example**

```java
clusterManager.upsertUser(
	AuthDomain.LOCAL,
	"alice",
	UserSettings.build()
		.password("password")
		.name("Alice Doe")
		.roles(new Role[]{ new Role("query_select","default"), new Role("fts_searcher", "default") })
)
```

  - Full path is based on the `AuthDomain`: `PUT /settings/rbac/users/local/alice`
  - For external, `PUT /settings/rbac/users/external/alice`
  - data payload is www-form-urlencoded: `"name=Alice Doe&roles=query_select[default],fts_searcher[default]&password=password"`
  - Roles should be concatenated to a single string and are comma separated. Client returns true if server responds OK, else false.

**One caveat**: the user is asynchronously updated on other servers in the cluster and `kv_engine`, this
should be properly documented in the API docs.

### `removeUser`
Deletes an Couchbase user. Note that depending on the `AuthDomain` this might be supported by the server
or not.

```
boolean removeUser(AuthDomain domain, String userid)
```

**Example**

```java
removeUser(AuthDomain.LOCAL, "alice")
```

 - For local, `DELETE /settings/rbac/users/local/alice`
 - For external, `DELETE /settings/rbac/users/external/alice`

### `getUsers`
Get all users from Couchbase based on the `AuthDomain`.

```
List<User> getUsers(AuthDomain domain)
```

A `User` is described with the following properties:

```
User {
	String name;
	String id;
	String domain;
	Role[] roles;
}

Role {
	String role;
	String bucket_name;
}
```

**Example**

```java
clusterManager.getUsers(AuthDomain.LOCAL)
```

The following shows an example server response to `GET /settings/rbac/users/local`:

```json
[
{
    "name": "Alice Doe",
    "id": "alice",
    "domain": "local",
    "roles": [
      {
        "role": "fts_searcher",
        "bucket_name": "default"
      },
      {
        "role": "n1ql_select",
        "bucket_name": "default"
      }
    ]
  }
]
```

 - For external, `GET /settings/rbac/users/external`


### `getUser`
Get user info for a particular Couchbase user.

```
User getUser(AuthDomain domain, String username)
```

The `User` instance is the same as described in `getUsers`.

**Example**

```java
clusterManager.getUser(AuthDomain.LOCAL, "alice")
```

Example response from `GET /settings/rbac/users/local/alice`:

```json
{
    "name": "Alice Doe",
    "id": "alice",
    "domain": "local",
    "roles": [
      {
        "role": "fts_searcher",
        "bucket_name": "default"
      },
      {
        "role": "n1ql_select",
        "bucket_name": "default"
      }
    ]
  }
```

 - For external, `GET /settings/rbac/users/external/alice`

## Language Specifics

### C
The object model of defining roles etc. is fairly complex for a C level API. Applications can manually access the RBAC API through `lcb_http3()` using `LCB_HTTP_TYPE_MANAGEMENT` as the type.

### .NET
.NET Does not use setting builders, so the implementation is done by adding an additional parameter for the AuthenticationDomain:  `UpsertUser(AuthenticationDomain domain, string username, string password, string name, params Role[] roles)`.

Additionally, there are implementations for async versions of for each of the overloads which take an AuthenticationDomain object.

AuthenticationDomain is an enum with the following values: `Local` and `External`

**Example:**

```net
var createResult = _clusterManager.UpsertUser(AuthenticationDomain.Local, user.Username, "secure123", user.Name, user.Roles.ToArray());
```

### PHP
In PHP API the functions which return indexes of the objects use "list" verb to make them distinguishable from singular form. Compare getDesignDocument/getDesignDocuments versus getDesignDocument/listDesignDocuments.

So PHP will use listUsers in this API, even though we don't have getUser function

```php
$userSettings = \Couchbase\UserSettings->build()
    ->password("password")
    ->role("data_reader", "myBucket");
$cluster->manager()->upsertUser("alice", $userSettings,
    \Couchbase\ClusterManager::RBAC_DOMAIN_LOCAL);
$cluster->manager()->removeUser("alice",
    \Couchbase\ClusterManager::RBAC_DOMAIN_LOCAL);
$cluster->manager()->listUsers(\Couchbase\ClusterManager::RBAC_DOMAIN_LOCAL);
```

### Node.js

```js
cluster->manager()->upsertUser(‘local’, ‘alice’, {
  password: ‘password’,
  roles: {role: ‘data_reader’, bucket: ‘myBucket’}
}, function(err) { })
cluster->manager()->removeUser(‘local’, ‘alice’, function(err) { });
cluster->manager()->listUsers(‘local’, function(err, users) { });
```

### Python
```python
Mgr = cluster.cluster_manager()
Users = cluster.users_get(AuthDomain.Local)
cluster.user_upsert(AuthDomain.Local, ‘mark’, ‘s3cr3t’, [(‘global-role’), (‘bucket-role’, ‘bucket-name’)])
cluster.user_remove(AuthDomain.Local, ‘mark’)

#Get_user = user_get
#Upsert_user = user_upsert
#Remove_user = user_remove
```

In Python it’s simpler to define a user and its roles using Python lists/tuples, with roles passed as discrete arguments rather than an actual object.

The methods exist as object-verb to be internally consistent with other Python management methods (some predating the RFC). Aliases to the common name are mentioned here as well for the sake of completeness.

### Java
```java
cluster.clusterManager().upsertUser(AuthDomain.LOCAL, “alice”, UserSettings.build().password(“password”)
       .roles(Arrays.asList(new UserRole("data_reader", “myBucket”))));
cluster.clusterManager().removeUser(AuthDomain.LOCAL, “alice”);
cluster.clusterManager().getUsers(AuthDomain.LOCAL);
```

### Go
```go
err := cluster.Manager(...).UpsertUser(gocb.LocalDomain, “alice”, gocb.UserSettings{
  Password: “password”,
  Roles: []gocb.UserRole{{“data_reader”, “myBucket”}},
})
err := cluster.Manager(...).RemoveUser(gocb.LocalDomain, “alice”)
users, err := cluster.Manager(...).GetUsers(gocb.LocalDomain)
```

## Additional Info
User roles info from ns_server. “*” as bucket_name implies access to all buckets.
https://github.com/couchbase/ns_server/blob/0038200eddba441182cf1743420cef78bad88029/src/menelaus_roles.erl

## Q&A
 1. I think we need to document the possible error cases for the methods above (what happens if user doesn’t exist and so forth as well and define the proper error messages for those cases)
 *Clients will throw exceptions similar to bucket management api. There is no special handling for user management.*
 2. Should we emulate the bucket management API a bit, like we have “hasBucket”, we could do a “hasUser” or similar that just loads all and checks if its in the list?
 *We are not doing this 100%, we will keep api simple. The main motivation is test scaffolding.*
 3. If the user already exists and we do an upsert without password, what is the expected behaviour?  Should it clear the password, or do we need to not send the password field?
 *Yes, don't send the password. It wouldn't clear the password if it is not updated.*
 4. How can this API consider that there are external users.
 *The scope of this RFC is not to manage those external users, however we should make sure that it’s possible.*

## Changelog

 - 03/10/2017 Rename deleteUser to removeUser similar to removeBucket
 - 03/10/2017 Rename getAllUsers to getUsers
 - 04/21/2017 Change rest endpoints to use local instead of builtin and getUsers response will include domain as internal instead of type.
 - 06/14/2017 Change endpoint for getUsers to fetch only internal users
 - 06/14/2017 Add getUser api to fetch a particular user’s info
 - 06/14/2017 Removed existing sign offs
 - 07/06/2017 Adding authentication domain

## Signoff

| Language | Representative | Date |
| - | - | - |
| C | Sergey Avseyev | 2017-08-29 |
| Go | Brett Lawson | 2017-07-26 |
| Java | Subhashni Balakrishnan| 2017-08-25 |
| .Net | Jeff Morris | 2017-09-06 |
| NodeJS | Brett Lawson | 2017-07-26 |
| PHP | Sergey Avseyev | 2017-07-26 |
| Python | Matt Ingenthron | 2017-08-29 |
