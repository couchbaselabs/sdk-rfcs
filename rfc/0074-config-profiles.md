# RFC-0074 Configuration Profiles

## Meta

* RFC Name: Configuration Profiles
* RFC ID: 0074-config-profiles
* Start Date: 2022-08-24
* Owner: Michael Reiche
* Current Status: Draft

## Summary

Specification for Configuration Profiles with configuration properties having different values than defaults.
From  https://issues.couchbase.com/browse/CBD-5045

## Motivation

The Default configuration properties are set following the Configuration specified in SDK 3 Foundation. https://github.com/couchbaselabs/sdk-rfcs/blob/master/rfc/0059-sdk3-foundation.md
* Configuration property values different from the defaults would be more appropriate for some environments.
* It is difficult to determine the values of the configuration properties when troubleshooting. 


## Limitations

The Configuration Profiles may property values may be an improvement upon the defaults for some cases.
They are not intended to be the optimimum values for all cases.

## Requirements

We want to introduce a feature that allows our SDKs to be quickly configured for some specific use-cases. The first profile we will create will be for "wan-development" environments and will change the default timeouts to work in high latency environments. However, the solution should be reusable to allow us to introduce additional profiles in the future (e.g., lambda/elixir may benefit from certain config defaults).
The solution must allow config values to be overwritten by values that are associated with a config profile. Some SDKs may be more flexible than others, but the bare minimum implementation must support the ability to overwrite the config values when a config profile is applied.
Also, it should be possible to enable logging of the values of the configuration properties that are being used by the SDK.

### Backwards compatibility
The existing mechanism and configuration property values will remain unchanged.

### New Configuration Profiles
* Sets of configuration property values will be defined by Configuration Profiles and referenced by a string name.
* Not all properties need to be specified in a profile.
* Properties that are specified in the configuration profile will overwrite properties in the environment object producing an environment object with the result.
* Properties that are not specified will be copied as-is.


### New Configuration Profile Mechanism
* A new Configurations Profile mechanism will allow new/different Configuration Profiles to be applied over existing configuration property values resulting in a new set of configuration property values (referred to as ClusterEnvironment in some SDKs).
* Subsequent Configuration Profiles can be applied to the new set of configuration property values.
* Configuration property values can be modified individually at any point before or after application of a Configuration Profile.


## Reference Design

This section offers recommendations for implementing.
Implementers should adhere to the reference design where it makes sense, and diverge where the idioms of a specific language or SDK provide a more natural solution.

### Generic Behavior

```
// Use the existing API to construct the environment (or environment builder depending on SDK)       
ClusterEnvironment originalEnv = …;                                                                  
                                                                                                     
// Optionally modify the resulting environment (or builder) with existing API                        
originalEnv.setXXXX(...)                                                                             
// Optionally apply a configuration profile to the original environment (or environment builder)     
ClusterEnvironment newEnv = applyProfile(originalEnv, "development");                                
                                                                                                     
// Optionally modify the resulting environment (or builder) with existing API                        
newEnv.setXXXX(...)                                                                                  
                                                                                                     
// Use the environment as needed                                                                     
Cluster cluster = Cluster.connect( … newEnv … );                                                     
```

### Implementation Suggestions
#### Java (uses a builder)

```java
ClusterEnvironment clusterEnv = ClusterEnvironment.builder()                                         
    .ioConfig(iocfg -> iocfg.numKvConnections(MY_NUM_KV_CONNECTIONS)) ← modify ioconfig              
    .applyProfile("development")  ← properties defined in "development" will overwrite               
    .ioConfig(iocfg -> iocfg.maxHttpConnections(MY_MAX_HTTP_CONNECTIONS)) ← modify ioconfig          
    .build();                                                                                        
```

#### SDKs that use a struct

```
ClusterEnvironment originalEnv = …                                                                   
newOptions.ioConfig.numKvConnections = MY_NUM_KV_CONNECTIONS;                                        
ClusterEnvironment newEnv = applyProfile(originalEnv, 'development')                                 
newOptions.ioConfig.maxHttpConnections = MY_MAX_HTTP_CONNECTIONS                                     
Cluster cluster = cluster.connect( … someEnv …)                                                      
```

**Note:** it won't be possible to implement profiles by representing the profile with the same data structure as the environment (or builder) and merging the profile onto the existing one - unless there is a way to represent the properties as not being assigned a value (for java, some of the configuration values are primitives (i.e. int) and would need to use a special value to mean unassigned (such as -1)).  Otherwise every property of the profile would appear to be assigned, and it would completely overwrite the initial configuration. 

## Logging
The configuration properties used by the SDK when connecting to the server should be logged to facilitate trouble-shooting.  This does not need to be in a standard format.  The Java SDK already does this as shown in the following sample:

```
INFO - {"timestamp":"2022-08-17 04:15:57.143","severity":"INFO","thread":"cb-events","class":"com.couchbase.core",
"crId":"","context":{"svcId":"NRTOVIGO","svcBld":"1","svcNm":"ordnrt_vigomtdata_v1_task"},
"msg":"[com.couchbase.core][CoreCreatedEvent] {"clientVersion":"3.3.0","clientGitHash":"${buildNumber}",
"coreVersion":"2.3.0","coreGitHash":"${buildNumber}",
"userAgent":"couchbase-java/3.3.0 (Linux 4.14.232-177.418.amzn2.x86_64 amd64; OpenJDK 64-Bit Server VM 13.0.6+5-Debian-1)",
"maxNumRequestsInRetry":32768,"ioEnvironment":{"nativeIoEnabled":false,"eventLoopThreadCount":8,
"eventLoopGroups":["NioEventLoopGroup"]},"ioConfig":{"captureTraffic":[],"mutationTokensEnabled":true...
```

## Profiles
Configurable Properties - from  SDK 3 Foundation RFC https://github.com/couchbaselabs/sdk-rfcs/blob/master/rfc/0059-sdk3-foundation.md. Note that only properties that are changed in the profile are included in this list (timeouts).

| name                      | type           | default (SDK3 Foundation)| wan-development |
|---------------------------|----------------|--------------------------|-----------------|
| KVConnectTimeout          | Duration       | 10s                      | 20s             |
| KVTimeout                 | Duration       | 2.5s                     | 20s             |
| KVDurableTimeout          | Duration       | 10s                      | 20s             |
| ViewTimeout               | Duration       | 75s                      | 120s            |
| QueryTimeout              | Duration       | 75s                      | 120s            |
| AnalyticsTimeout          | Duration       | 75s                      | 120s            |
| SearchTimeout             | Duration       | 75s                      | 120s            |
| ManagementTimeout         | Duration       | 75s                      | 120s            |

## Future Extensions
Possibly maintain profiles in an external source (classpath resource, file etc) so that the values can be modified without changing code,  or that new profiles can be created. 
For spring-boot applications, there is already a number of mechanisms to externalize properties which an application could leverage if so inclined - https://docs.spring.io/spring-boot/docs/3.0.0-M4/reference/html/features.html#features.external-config ]


## Changelog
* Sep 19, 2022 - Revision #2 (by Michael Nitschinger)
    * Formatting Polish, Signoff
* Aug 24, 2022 - Revision #1 (by Michael Reiche)
    * Version Draft 

## Signoff

| Language     | Team Member        | Signoff Date | Revision |
|--------------|--------------------|--------------|----------|
| C            | Sergey Avseyev     | 2022-09-02   | #1       |
| Connectors   | David Nault        | 2022-09-01   | #1       |
| Go           | Charles Dixon      |              |          |
| Java         | Michael Nitchinger | 2022-09-19   | #2       |
| .NET         | Jeffry Morris      | 2022-09-08   | #1       |
| Node.js      | Brett Lawson       |              |          |
| PHP          | Sergey Avseyev     | 2022-09-02   | #1       |
| Python       | Jared Casey        | 2022-09-02   | #1       |
| Ruby         | Sergey Avseyev     | 2022-09-02   | #1       |
| Scala        | Graham Pople       | 2022-09-08   | #1       |



