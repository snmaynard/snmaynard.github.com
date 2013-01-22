--- 
layout: post
title: Redis Masterclass - Part 2, Monitoring
---

This is the second in my three part series about using Redis. The first post was about [Configuring Redis](/2013/01/14/redis-masterclass-part-one-configuring-redis).

I've seen monitoring Redis instances at scale at both [Bugsnag](https://bugsnag.com), and [Heyzap](https://heyzap.com) and I thought I'd share what I think should be monitored on a Redis instance.

## Monitoring

Monitoring Redis instances, especially if you shard and have many instances of Redis, can be a little intimidating. But Redis is pretty easy to monitor with the [INFO](http://redis.io/commands/info) command. You can easily script any checks by running the command `redis-cli info`.

### Used Memory

As Redis stores everything in memory, you really need to monitor your data size. If you start to use more than the available RAM, the [out of memory process killer](http://linux-mm.org/OOM_Killer) is likely to kill Redis, which will cause downtime. Also, the worst kind of downtime where it will take significant time for you to come back up while Redis reloads its data in to memory, if you can even fit it in memory.

You should monitor both `used_memory` and `used_memory_peak` from within the INFO command output. Ideally you should chart these values and set up alerts to give you plenty of headroom. You should also know what you are going to do if you hit that threshold. You could resize your slave to have more memory, clear out some old keys or move an instance to a new machine for example.

### Persistence

You should monitor the persistence of your Redis dataset. If there is a problem with the machine, or the Redis instance crashes then the in memory dataset is lost and the data will be loaded from disk. You can monitor when the data was last written to disk by checking the `rdb_last_save_time`. You can also monitor how many changes would be lost if the instance was to crash by checking `rdb_changes_since_last_save`.

I've seen a lack of persistence monitoring cause significant data loss before, and due to the nature of the data loss, its permanent. No backups exist without persistence, so make sure that if you want persistent data that you monitor this.

### Replication

If you have a Redis slave, you need to ensure that the replication is still working by checking the `master_link_status` value. If this is `up`, then the replication is working. If the value is 'down' then the INFO output will have some more diagnostic information for you,

{% highlight javascript %}role:slave
master_host:192.168.1.128
master_port:6379
master_link_status:down
master_last_io_seconds_ago:-1
master_sync_in_progress:0
master_link_down_since_seconds:1356900595{% endhighlight %}

### Fork Performance

When forking to save the dataset to disk, the Redis instance will be blocked and no commands can be run. You can monitor how long the fork takes by monitoring the value of `latest_fork_usec`. This is how long the Redis instance was blocked for and was unable to execute any commands. This can be hundreds of milliseconds per GB of memory usage running on a VM for example.

This can be negated by using a redis slave for persistence, and having the master not persist. That way you get a backup without having to pay the price of forking on your master. You need to ensure that replication is monitored if you use this option.

### Redis Config Consistency

Redis provides functionality to change the configuration of a running instance by running the [CONFIG SET](http://redis.io/commands/config-set) command. However, this command does not change the redis.conf file, so any restart of the instance will mean that the configuration will revert to what is contained in the redis.conf file. You should monitor the consistency of the Redis config by comparing the output of the [CONFIG GET](http://redis.io/commands/config-get) command with the redis.conf file. Any difference should alert until the config file is updated.

### Slow Query Log

Redis provides a [SLOWLOG](http://redis.io/commands/slowlog) command that allows you to query the slow query log. This log is in memory, so there is no log file to monitor (you have to use redis-cli), but it also means that the performance hit of enabling the log is very low. I like to have this enabled with a relatively high query time. I then monitor the contents of the slow log using a cron job, outputting to a log file and use [Kibana](http://kibana.org/) to visualize the slow queries.

One caveat of the slow query log, is that it only accounts for the time to run the query itself. No network latency is included, so your app may be waiting significantly longer for a query to complete if that is the source of your problems.

Slow queries on Redis will be different to other database systems you are used to. Due to the single threaded nature of Redis, only one query can be run at a time. So queries taking 10 ms may be more of a problem than you are used to. Thankfully Redis is blisteringly fast in general, so this problem usually doesn't appear. If it does, then you can shard to get greater parallelism, or try to change the datastructure or query so that Redis has less work to do.

### Monitoring Apps

#### Sentinel

[Sentinel](http://redis.io/topics/sentinel) is an official replication monitoring tool. It will check your replication is working as expected, and if it finds an issue can warn you or even automatically failover to a slave.

#### Redis Live

[Redis Live](http://www.nkrode.com/article/real-time-dashboard-for-redis) is a more general Redis monitoring solution. It periodically runs [MONITOR](http://redis.io/commands/monitor) to monitor which commands are running against an instance and provides some pretty useful analytics for Redis. The output is to a webpage that provides some nice visualizations of the output.

#### Redis Faina

[Redis Faina](http://instagram-engineering.tumblr.com/post/23132009381/redis-faina-a-query-analysis-tool-for-redis) is another, lightweight alternative to Redis Live from Instagram. It provides similar results to Redis Live, monitoring the [MONITOR](http://redis.io/commands/monitor) command for slow commands and shows your command distribution.

The one issue with faina is mentioned in the instagram docs,

<em>One caveat on timing: MONITOR only shows the time a command completed, not when it started. On a very busy Redis server (like most of ours), this is fine because there’s always a request waiting to execute, but if you’re at a lesser rate of requests, the time taken will not be accurate.</em>

If this applies to you, the [SLOWLOG](http://redis.io/commands/slowlog) command may be more applicable.

## Key Distribution

Trying to get visibility of key distribution in a Redis instance can be challenging. Especially trying to see which set of keys is using the most memory. There are some tools out there which take a look at your Redis keys and give you some information about the distribution.

### Redis-sampler

[Redis sampler](https://github.com/antirez/redis-sampler) is written by Antirez himself (one of the maintainers of Redis) and allows you to sample your Redis keys to get some stats about the sizes of strings stored in various keys. It also provides some statistical analysis around key usage. Its pretty useful when trying to get an idea of key types and how many of each you have.

### Redis-audit

[Redis audit](https://github.com/snmaynard/redis-audit) is a script I wrote in order to group similar looking keys together and to show you an approximation for their memory usage. It provides you information about how often certain groups of keys are used, how many of them have expiries set and how much memory they are using. I've found this useful to track down keys which never expire as well as looking for unused groups of keys.

### Redis-rdb-tools

[Redis-rdb-tools](https://github.com/sripathikrishnan/redis-rdb-tools) is similar to Redis-audit, but instead works on the rdb file that Redis dumps when persisting. It outputs a csv of all the keys in the rdb file, so you can run your own analysis on them.