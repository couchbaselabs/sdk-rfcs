# Meta

 - RFC Name: Fast-Failover SDK
 - RFC ID: 24
 - Start Date: 2017-03-03
 - Owner: Jeffry Morris
 - Current Status: ACCEPTED

# Summary
Fast-failover is a new feature coming with the Couchbase Server "Spock" version (5.0). It allows for a configurable setting on the server side for determining how quickly the server will automatically failover a node. This sdk-rfc defines the method of discovery and recovery for a failed over node from the the perspective of an SDK.

This sdk-rfc covers scenarios for Query, View, FTS, Analytics and K/V services and any combination of these services enabled on a Couchbase node.

The scope of this document is limited to `couchbase` and `ephemeral` buckets which will be supported in the upcoming 5.0 release.

# Motivation
Couchbase Server offers high-availability as a core feature; the current automatic failover settings can be configured as low as 30 seconds. With a 30 second automatic failover setting an application could take as much as 120 seconds to detect the failover which would mean the application might be down for at least that amount of time.

With fast-failover, that time can be reduced to as little as a 5 seconds on the server.  In practice, that means failure may be declared after 5 seconds, but failover may take slightly longer. In turn, the SDK must be able to identify when a node may be down as quickly as possible and reconfigure itself to the updated cluster-map. Additionally, all SDKs should follow similar rules for identifying when a node is down and be consistent in their behavior and implementation (as much as possible barring platform idiosyncrasies).

# Current Implementations
## .NET
The .NET SDK uses two means of determining if a failover has occurred and/or whether a node is unreachable. One is a means of determining if a node is no longer responsive (connections dropped, cannot reconnect, etc), the other is a check and compare of the server config for each active node with the SDK’s current configuration.

In the former, if `x` number of failures occur within a timespan `y`, then the node will be put into a `dead` state. Any K/V operations mapped to this node will result in a `NodeUnavailableException` being returned in the Exception property of the `IResult` implementation; N1QL and View requests will be routed towards other active nodes. Once a node is considered dead, then a timer will fire every 1s in an attempt to create a new connection and execute a NOOP against it. If the node responds, the node will become active again. If not, the timer will continue to fire until successful. The point of this is to keep the client from tieing up the application threads waiting for operations that will end up failing at the network level and instead just fail fast with an error until a connection can be made with the node.

For the latter, there is a timer that fires every 10s which requests a config from each available (not dead) node. The revision is compared against the current cluster configuration that the client is using. If the revision has changed, then the client will be updated with the new configuration.

## Java
In Java `core-io` we detect node failures through the following mechanisms in CCCP mode:

 - If an endpoint (socket) goes into a `DISCONNECTED` state we’ll proactively fetch a new configuration via `GET_CONFIG` from the cluster.
 - If the server returns a NMVB we take that config and apply it
 - Every 10 seconds, we also poll for a new config via `GET_CONFIG` from one of the nodes in the cluster (this is meant as a backup to grab a new config even if missed by the other two reactive approaches)
As another proactive measure under CCCP when a config is marked as tainted (that is under rebalance when a FF-Map is available) we start polling every second for a new config until the rebalance finished (config is not marked tainted anymore).

For HTTP mode (if CCCP is not available/disabled), we have the streaming conn attached anyways, so we still do the NMVB override but both `GET_CONFIG` approaches (on disconnect & every 10s as well as the rebalance tainted polling) are essentially a `NOOP`.

## LCB
LCB does not do background polling (it could, but it wouldn’t be useful for anything except node.js and other “true” async use cases).

LCB has a default throttling of 10s: Whenever a network error occurs, lcb triggers a config request. This will cause it to query a node if no other config request has been performed in the prior 10 seconds.

The config request takes place in a different subsystem, and varies on the config transport being used:

 - When using “CCCP” it will merely request from the next node in the list, with an index variable wrapping around. If the server does not respond within 2.5s (or some other error occurs), the config is requested from the next node.
 - When using HTTP it will expect a new config over the stream of the current ‘entry point node’; if there is no config pushed within 2.5s (configurable too!) it will assume that the node is down and request a config from the next node, which will effectively become the EP node.

The throttle interval is configurable.

### LCB - Modifications
 - Lower the refresh-on-error ‘floor’ to 10ms from 100s.
 - “Background-poll” every 2.5 seconds. For clients that are synchronous, this will still poll every 2.5 seconds, or whenever the client is next active again -- depending on how active the client is. For clients that are async like node.js, it will behave similar to other clients’ behavior in the RFC.

## GO
In gocbcore we detect node failures through the following mechanisms in CCCP mode:

 - If the server returns a NMVB we take that config and apply it
 - Every 10s seconds, we also poll for a new config via `GET_CONFIG` from one of the nodes in the cluster

For HTTP mode (if CCCP is not available/disabled), we have the streaming conn attached anyways, so we still do the NMVB override but both `GET_CONFIG` approaches (every 10s) are essentially a `NOOP`.

## PHP, Python, NodeJS
Since they are language bindings, the implementation is handled by LCB (see above).

# General Design
For Fast-Failover to ­­­­be effective, the client SDK must quickly determine if a config change has occurred. Server configurations are acquired by either a) explicitly requesting them using the Config Command (`0x05`) or b) as the body of a K/V operation that has failed with a NMVB status. Typically a configuration is requested by an event within the application such as an IO error or by polling every at set intervals. Once a configuration has been obtained, the revision should be compared against the current configuration and if greater, swapped in the client, along with the VBucket mapping and connections if cluster endpoints have been added or removed.

For example, in a centralized location the following methods exist:

```
bool CompareAndSwap(config){
	lastCheckedTime = Time.Now;

  if(config.Rev >  this.currentConfig.Rev) {
  	exchange(config, this.currentConfig);
  	return true;
  }
  return false;
}

void CheckConfigs(excluded) {
  If(lastCheckedTime + ConfigPollFoorInterval < Time.Now()) {
    foreach(server in cluster) {
      If(server != excluded) {
        var config = new ConfigRequest();         	
        server.Send(config);
        CompareAndSwap(config);
        break;
      }
    }
  }
}
```

In the case of an IO error or NMVB, the following pseudo code would exist at the IO layer of an SDK:

```
result Send(operation) {
  Try {
    Connection.Write(operation.GetBytes());
    Operation.ReadBytes( Connection.Read());
  } catch(IOException) {
    CheckConfigs(this);
  } catch(NMVException) {
    CompareAndSwap(operation.GetConfig());
  }
  return Operation.Result;
}
```

For continuous polling for config changes, the following pseudocode illustrates:

```
startIndex = -1;
while (true) {
  Time.Sleep(ConfigPollInterval);
  if (startIndex < 0){
    startIndex = rand(_currentConfig.NodeCount);
  }

  for (i=0; i<_currentConfig.NodeCount; i++) {
    index = (startIndex + i) % _currentConfig.NodeCount;
    server = _currentConfig.GetServer(index);

    if (server.LastCheckedTime + ConfigPollFoorInterval < time.Now) {
      newConfig = server.Send(new ConfigRequest());
      if (CompareAndSwap(newConfig())) {
				break;
			}
	  }        	
  }
}
```

The minimum configurable failover detection interval is 5s on the server. If we chose a configurable, default polling interval of 2.5s then we should roughly cover the minimum Fast-Failover configuration from the server. However, to minimize chatter, a tunable property with a default floor of 50ms must be implemented - a user could change the tunable to a lower value like 2.5s if necessary, but it should not go below the floor defined. The trade-offs here **must be documented**.

```                                                                                                                                                     

 SDK poll,              SDK poll,            SDK poll,                     Failover                SDK poll,
 no change              no change            no change    Auto Failover    complete                no change
                                  SDK poll,                  Timeout                 SDK adjust to
     │    Failure occurs   │      no change     │       (failover detected)   │      new topology      │
     │                     │                    │       ┌─>           ┌─>     │                        │
     ├──────────┬──────────┼─────────┬──────────┼───────┼─────┬───────┼───────┼───────────┬────────────┤
     │          │          │         │          │      ─┘     │      ─┘       │           │            │
     │          │          │         │          │             │               │           │            │

    0s          1s         5s       10s        15s           16s             17s         20s          25s
 
                                                                                                                                                        
```

Above is a timeline showing the events that occur with a Fast-failover time of 15 seconds on the server and a client polling every 5 seconds for a new config revision. In this case, the failover occurs at 1 second and is detected by the cluster at 16 seconds and the failover completes at 17 seconds (assuming it takes 1 second for the failover to occur). At the 20 second mark, the client will detect the new config revision and reconfigure itself to the new topology. By reducing the server side Fast failover value, and adjusting the frequency of the client-side polling interval, the amount of time for a failure to occur, be detected and for the client to adjust can be minimized.

Since we want to minimize blocking, platforms that provide multi-thread or some kind of parallel IO support should use a separate worker thread or equivalent means of offloading the polling from main processing.

Once we have gotten a cluster map, the revision should be compared with the client’s current revision. If the revision is newer, the client should re-configure itself with the newer configuration to reflect the current server topology. Note that the polling should stop at the first config found and not continue through checking the configs over the entire cluster.

# Conclusion
Since we want to discover a cluster map change ASAP and because a cluster configuration can come in many flavors, a hybrid strategy for failover detection is probably our best bet. The client should react to errors related to connectivity (host not found, timeouts, etc) by checking for a config update when such errors occur; additionally, the client should implement a cluster map poll mechanism for detecting cluster map changes concurrently. In both cases the comparison algorithm (old vs new rev) should be the same and so should the reconfiguration process within the SDK.

To ensure that we don’t spam the server with config requests from polling and pulling, the same timestamp checking for floor and ceiling should be used as introduced in the “K/V Pulling” section above. The floor for config checking should be some value such as 50ms where we skip a check if a config request has already happened within that time. The ceiling should be a larger value such as 2.5s - both values should be tunable via client configuration.

# Addendum
The following configuration properties and default values are defined - note that all must be tunable:

| Name                         | Default Value |
| ---------------------------- | ------------- |
| ConfigPollInterval           | 2.5s          |
| ConfigPollFloorInterval      | 50ms          |

Note that these values must be tunable. The individual SDK naming conventions for configuration names may be applied.

# Questions
No questions recorded.

# Changelog
 - 7/6/2017: Initial draft publication
 - 10/3/2017
	- Change ConfigCheckInterval to ConfigPollInterval
	- Change ConfigCheckFloorInterval to ConfigPollFloorInterval


# Signoff
If signed off, each representative agrees both the API and the behavior will be implemented as specified.

| Language | Representative | Date       |
| -------- | -------------- | ---------- |
| Java     | Michael N.     | 2017-07-27 |
| .NET     | Jeff M.        | 2017-08-17 |
| Node     | Brett L.       | 2017-08-09 |
| PHP      | Sergey A.      | 2017-07-26 |
| Python   | Matt I.        | 2017-08-25 |
| Go       | Brett L.       | 2017-08-09 |
| C        | Sergey A.      | 2017-07-26 |
