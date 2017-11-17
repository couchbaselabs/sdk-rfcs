# Meta
- RFC Name: Ketama Hashing
- RFC ID: 26
- Start Date: 2016-12-21
- Owner: Mike Goldsmith
- Current Status: Accepted

# Summary
When storing documents in a Memcached bucket, the SDKs use a Ketama hashing algorithm to generate a ring of servers that can be used in to locate a server given a document key to create a robust and evenly distributed list of servers in the cluster.

# Hashed Values
The server hashes are MD5 hashes constructed using a server's IP, port and a repetition value, in the form:

    <ip>:<port>-<repetition>

For example:
```
127.0.0.1:8091-0
127.0.0.1:8091-1
127.0.0.1:8091-2
```

# Repetitions
40 hashes for each server in the cluster with the data server are generated.

# Verification Process
Add unit test to ensure a four node cluster generates the correct hashes that point to the correct server. The following JSON file contains a list of hash and hostname combinations that each SDK is expected to produce. The SDK should read the contents of the file and compare the generated hashes.

Hostnames:
```
192.168.1.101:11210
192.168.1.102:11210
192.168.1.103:11210
192.168.1.104:11210
```

[Expected Hashes](ketama-hashes.json)

# Code Examples
## .NET
```csharp
using (var md5 = MD5.Create())
{
    foreach (var server in _servers.Values.Where(x => x.IsDataNode))
    {
        const long repititions = 40;
        for (long rep = 0; rep < repititions; rep++)
        {
            var bytes = Encoding.UTF8.GetBytes(string.Format("{0}-{1}", server.EndPoint, rep));
            var hash = md5.ComputeHash(bytes);
            for (var j = 0; j < 4; j++)
            {
                var key = ((long) (hash[3 + j * 4] & 0xFF) << 24)
                          | ((long) (hash[2 + j * 4] & 0xFF) << 16)
                          | ((long) (hash[1 + j * 4] & 0xFF) << 8)
                          | (uint) (hash[0 + j * 4] & 0xFF);

                _buckets[key] = server;
            }
        }
    }
}
```

## Java
```java
private void populateKetamaNodes() {
    for (NodeInfo node : nodes()) {
        if (!node.services().containsKey(ServiceType.BINARY)) {
            continue;
        }

        for (int i = 0; i < 40; i++) {
            MessageDigest md5;
            try {
                md5 = MessageDigest.getInstance("MD5");
                md5.update(env.memcachedHashingStrategy().hash(node, i).getBytes(CharsetUtil.UTF_8));
                byte[] digest = md5.digest();
                for (int j = 0; j < 4; j++) {
                    Long key = ((long) (digest[3 + j * 4] & 0xFF) << 24)
                        | ((long) (digest[2 + j * 4] & 0xFF) << 16)
                        | ((long) (digest[1 + j * 4] & 0xFF) << 8)
                        | (digest[j * 4] & 0xFF);
                    ketamaNodes.put(key, node);
                }
            } catch (NoSuchAlgorithmException e) {
                throw new IllegalStateException("Could not populate ketama nodes.", e);
            }
        }
    }
}
```

## libcouchbase
```c
static int
update_ketama(lcbvb_CONFIG *cfg)
{
    char host[MAX_AUTHORITY_SIZE+10] = "";
    int nhost;
    unsigned pp, hh, ss, nn;
    unsigned char digest[16];
    lcbvb_CONTINUUM *new_continuum, *old_continuum;

    qsort(cfg->servers, cfg->ndatasrv, sizeof(*cfg->servers), server_cmp);

    new_continuum = calloc(160 * cfg->ndatasrv, sizeof(*new_continuum));
    /* 40 hashes, 4 numbers per hash = 160 points per server */
    for (ss = 0, pp = 0; ss < cfg->ndatasrv; ++ss) {
        /* we can add more points to server which have more memory */
        for (hh = 0; hh < 40; ++hh) {
            lcbvb_SERVER *srv = cfg->servers + ss;
            nhost = snprintf(host, MAX_AUTHORITY_SIZE+10, "%s-%u", srv->authority, hh);
            vb__hash_md5(host, nhost, digest);
            for (nn = 0; nn < 4; ++nn, ++pp) {
                new_continuum[pp].index = ss;
                new_continuum[pp].point = ((uint32_t) (digest[3 + nn * 4] & 0xFF) << 24)
                                        | ((uint32_t) (digest[2 + nn * 4] & 0xFF) << 16)
                                        | ((uint32_t) (digest[1 + nn * 4] & 0xFF) << 8)
                                        | (digest[0 + nn * 4] & 0xFF);
            }
        }
    }

    qsort(new_continuum, pp, sizeof *new_continuum, continuum_item_cmp);
    old_continuum = cfg->continuum;
    cfg->continuum = new_continuum;
    cfg->ncontinuum = pp;
    free(old_continuum);
    return 1;
}
```

## Go
```go
func newKetamaContinuum(serverList []string) *ketamaContinuum {
	continuum := ketamaContinuum{}

	// Libcouchbase presorts this. Might not strictly be required..
	sort.Strings(serverList)

	for ss, authority := range serverList {
		// 160 points per server
		for hh := 0; hh < 40; hh++ {
			hostkey := []byte(fmt.Sprintf("%s-%d", authority, hh))
			digest := md5.Sum(hostkey)

			for nn := 0; nn < 4; nn++ {

				var d1 = uint32(digest[3+nn*4]&0xff) << 24
				var d2 = uint32(digest[2+nn*4]&0xff) << 16
				var d3 = uint32(digest[1+nn*4]&0xff) << 8
				var d4 = uint32(digest[0+nn*4] & 0xff)
				var point = d1 | d2 | d3 | d4

				continuum.entries = append(continuum.entries, routeKetamaContinuum{
					point: point,
					index: uint32(ss),
				})
			}
		}
	}

	sort.Sort(ketamaSorter{continuum.entries})

	return &continuum
}
```

# Signoff

| Language | Representative | Date |
| - | - | - |
| C | Mark Nunberg | 21st March 2017 |
| Go | Brett Lawson | 22nd Marcg 2017 |
| Java | Michael Nitschinger | 21st March 2017 |
| .NET | Jeff Morris | 23rd March 2017 |
| NodeJS | Brett Lawson | 22d March 2017 |
| PHP | Sergey Avseyev | 27th March 2017 |
| Python | Mark Nunberg | 21st March 2017 |
