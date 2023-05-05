---
layout: post
title: "Problems regarding so-called 'Big Key' in redis"
# subtitle: "This is the post subtitle."
comments: true
date:   2023-04-22 21:11:35 +0800
categories: 'en'
---

### Big Key Definition

Basically key is considered as big key if it has either too much memory usage or too many elements.

- A STRING key whose size of value is 5 MB (the data is too large).
- A LIST key with 20,000 elements (the number of elements in the list is excessive).
- A ZSET key with 10,000 members (the number of members is excessive).
- A HASH key whose size is 100MB even if only contains 1,000 members (the size of the key is too large). 

It should be noted that the definition of Big Key might be different according to actual use cases and business scenarios of Redis. This is to say that you should judge taking all factors into consideration.[^fn3]

### Discover Big Keys

1. Use `redis-cli --bigkeys` or `redis-cli --memkeys` to detect big keys [^fn1]
2. Use `memory usage <key>` to find the memory usage of speficied key, careful when using in production environment as it may block other commands[^fn2]

### Typical Problems

- **Execution slower**: command executed much slower than others[^fn4], typically the execution time should under 1ms, and the default slowlog config is 10ms. Slow execution may block other cmds and even result in timeout for clients.

- **Key eviction**: if maxmemory and eviction policy is set, big keys get evicted(depends on the policy) which need much time to re-fill cache later

- **Network latency**: if big key is frequently queried by clients, it will cause a significant amount of network traffic and may result in network congestion

- **Backup impact**: AOF in redis has three modes of persistence: no, everysec, always. [^fn5] In 'always' mode, redis main thread will wait until fsync completed, big key will result in much more blocking time [^fn7] (however, 'always' is not a default nor a recommended config, it shouldn't be a problem in most cases)

- **Read/Write skew**: in redis cluster, keys are sharded in different nodes, frequent read/write to big keys may result in unbalanced resource usage, which means some nodes may have much higher work load

### Appendix

Two kinds of object encoding for HASH: ziplist(listpack), hashtable. [^fn6]

Use `object encoding <key>` to detect the actual encoding type.`

In general, ziplist or listpack is more memory efficient than hashtable but the time complexity for example `HGET`, `HSET` is O(n), since the elements size is limited, it won't cause performance issue, however, as the size grows the time complexity will not acceptable or as the entries exceeds memory limit than the compression will not bring much benefits any more.

### References
[^fn1]: https://redis.io/docs/ui/cli/#scanning-for-big-keys
[^fn2]: https://redis.io/commands/memory-usage/
[^fn3]: https://www.alibabacloud.com/blog/a-detailed-explanation-of-the-detection-and-processing-of-bigkey-and-hotkey-in-redis_598143
[^fn4]: https://developer.redis.com/operate/redis-at-scale/observability/identifying-issues/
[^fn5]: https://redis.io/docs/management/persistence/
[^fn6]: https://redis.com/ebook/part-2-core-concepts/01chapter-9-reducing-memory-use/9-1-short-structures/9-1-1-the-ziplist-representation/
[^fn7]: https://twitter.com/alexxubyte/status/1571880260177907713?lang=en
[^fn8]: https://redis.io/docs/management/optimization/memory-optimization/#using-hashes-to-abstract-a-very-memory-efficient-plain-key-value-store-on-top-of-redis