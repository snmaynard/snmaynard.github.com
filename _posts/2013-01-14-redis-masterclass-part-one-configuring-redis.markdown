--- 
layout: post
title: Redis Masterclass - Part 1, Configuration
---

I've been using Redis for the past couple of years, we use it extensively at [Bugsnag](https://bugsnag.com) (app-error monitoring service) and I've grown to love it. It is the datastore I trust by far the most, with very good reliability and performance characteristics.

Some good tips on configuring and using Redis are available on the [Redis website](http://redis.io/topics/admin). If you are using Redis in production or are thinking about it then I recommend looking through the Redis site, as it has very good [documentation](http://redis.io/documentation). Here is a selection of the things I think are most important (your use case may vary).

## Limit Redis Memory Usage

Redis is an in memory datastore. If we let its memory usage go unchecked, we could run out of memory on the machine and the [out of memory process killer](http://linux-mm.org/OOM_Killer) could kill the Redis process, causing downtime. I've seen this happen at scale, and it takes an agonising amount of time for redis to restart and reload its data from disk, so we should avoid this at all costs!

You can set the maximum amount of memory an instance of Redis should use by setting the [maxmemory](https://github.com/antirez/redis/blob/2.6/redis.conf#L258) configuration setting. From the [docs](http://redis.io/topics/faq)

> If this limit is reached Redis will start to reply with an error to write commands (but will continue to accept read-only commands), or you can configure it to evict keys when the max memory limit is reached in the case you are using Redis for caching.

To configure it to evict keys you can set the [maxmemory-policy](https://github.com/antirez/redis/blob/2.6/redis.conf#L283). There are a lot of different algorithms for selecting a key to evict that you can choose from. Some will suit a Redis instance that is used as an LRU cache, while others are better for instances that have a mix of cached values.

## Enable Overcommit Memory

You should ensure that [overcommit memory](http://www.redhat.com/magazine/001nov04/features/vm/) is set to 1 in linux. This ensures that malloc will allow more memory than the machine has available to be allocated, under the assumption that not all allocated memory will be written to. This is the case when a process forks, like Redis, due to [Copy-on-write](http://en.wikipedia.org/wiki/Copy-on-write).

I've seen this disabled by mistake on a running Redis instance and all background saves of the Redis dataset started failing, so this is really important. This meant we were running without any backups for data stored in Redis, so there were a few hours of panic diagnosing the issue!

## Ensure Configuration Consistency

To configure Redis, you can use the [redis.conf](https://github.com/antirez/redis/blob/2.6/redis.conf) config file or you can use the [CONFIG SET](http://redis.io/commands/config-set) command to adjust the configuration of a running Redis instance without restarting it. When a Redis instance first starts, it reads its config from the redis.conf file.

If you use the CONFIG SET command to change the running configuration of the instance, the redis.conf file remains the same. Which means that any future restart of the Redis instance will read the old configuration file. I have seen this cause downtime in production environments more than once, and at Heyzap we had monitoring installed to verify that the running configuration was the same as the redis.conf file. If you want to monitor this yourself, you can check the output of the [CONFIG GET](http://redis.io/commands/config-get) command.

## Enable Swap

Configure swap. This will at least prevent the OOM killer from killing Redis. If you spill in to swap, you get some breathing room to come up with a strategy to fix it, rather than trying to start a Redis instance that won't fit into RAM when it starts. From the [docs](http://redis.io/topics/admin),

>Make sure to setup some swap in your system (we suggest as much as swap as memory). If Linux does not have swap and your Redis instance accidentally consumes too much memory, either Redis will crash for out of memory or the Linux kernel OOM killer will kill the Redis process.

**WARNING:** When using swap on a Redis instance be sure to monitor that the swap is not used by Redis. Your server will dramatically slow down if reading keys stored in swap.

## Use Sharding

Use multiple instances of Redis from the start, even if initially only on a single machine. There are many advantages to sharding, and very little in the way of reasons not too. There is a useful discussion of sharding Redis [here](http://oldblog.antirez.com/post/redis-presharding.html), but here's my take.

### Advantages Of Sharding

#### Quickly Move Instances

Moving an instance of Redis from one machine to another will immediately help alleviate memory issues you have on one of your Redis machines. If you have a single Redis instance, this can take much longer and will involve more application code changes, rather than just simply updating the IP address of the instance in question.

#### Faster Fork Times

In order to save the data in the Redis database to disk for persistence, Redis will fork to persist the data to disk while the original process continues serving requests. The action of forking a new process will block the original process, which means that requests are not being processed until the fork has completed.

Forking is a non trivial operation, and the fork time is proportional to the memory usage of the process being forked. [Fork times](http://redis.io/topics/latency) can be hundreds of milliseconds per GB, depending on the system you are running on. Having more instances, each with a smaller data size, will help with this issue.

#### Greater Parallelism

Redis is single-threaded by design. This means that each instance of Redis can only execute one command at a time. This is useful for atomic updates, and simplifies a lot of client code and transactions. This also means that when a command being run on a Redis instance is slow, it will block all other commands from running.

If you have many instances of Redis however, this is less of an issue as your other instances of Redis will be unaffected by the slow command. Some people may think that Redis' support of multiple databases are a good approximation to sharding, but unfortunately all databases run in the single thread. Meaning that a blocking command on one database in an instance blocks all other databases from having commands run against them.

#### More Configuration Flexibility

If you use multiple instances, say one is a pure [LRU cache](http://en.wikipedia.org/wiki/Cache_algorithms#Least_Recently_Used) and another has persistent application data in. You can configure each instance of Redis to deal with hitting its maxmemory limit differently. In an LRU cache, removing the LRU key when hitting the maxmemory limit would be sensible. However in an instance with persistent application data in, you may want writes to be denied, but reads to continue. Having multiple instances allows you to have this configuration flexibility.


### Disadvantages Of Sharding

The disadvantages of sharding are fairly weak in my opinion, but included here for completeness.

#### More Initial Effort

Having more Redis instances means you have to configure multiple Redis instances, even if initially you don't need them. Monitoring those instances will also be more effort as well.

#### Greater Memory Usage

Each instance of Redis comes with its own memory overhead. An instance of Redis running without any keys uses under 1 MB of memory. 

### Sharding Strategies

When sharding your Redis instances you need a way of splitting the keys between different instances. At [Bugsnag](https://bugsnag.com) and [Heyzap](https://heyzap.com) keys are sharded predictably on the client, so we can still [SUNION](http://redis.io/commands/sunion) two sets in the same instance where we need to. This means we generally shard by feature. For example, grouping all sets of users following other users in the same redis allows us to quickly find the common followers for two users. This has the added advantage of being able to see how much Redis memory is being used by a given feature.

There are a lot of ways of sharding Redis keys, but be aware that if two keys are on different instances of Redis, you can't atomically update both simultaneously. You can read about sharding strategies in the [Redis documentation](http://redis.io/topics/partitioning).

## Part Two - Monitoring

Part Two of the series will cover how to monitor your Redis instances. Look out for it in the next few days.