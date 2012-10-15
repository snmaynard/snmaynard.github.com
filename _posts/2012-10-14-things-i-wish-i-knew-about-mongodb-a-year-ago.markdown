--- 
layout: post
title: Things I wish I knew about MongoDB a year ago
---

I've used MongoDB for over a year at scale at both [Heyzap](http://www.heyzap.com) and [Bugsnag](https://bugsnag.com) and along the way I've learnt many things the hard way. Here is a summary of the things I wish someone had told me earlier.

## Selective counts are slow even if indexed

For example, when paginating a users feed of activity, you might see something like,
    
{% highlight javascript %}db.collection.count({username: "my_username"});{% endhighlight %}

In MongoDB this count can take orders of magnitude longer than you would expect. There is an [open ticket](https://jira.mongodb.org/browse/SERVER-1752) and is currently slated for 2.4, so here's hoping they'll get it out. 
Until then you are left aggregating the data yourself. You could store the aggregated count in mongo itself using the [`$inc` command](http://www.mongodb.org/display/DOCS/Updating#Updating-%24inc) when inserting a new document.

## Inconsistent reads in replica sets

When you start using replica sets to distribute your reads across a cluster, you can get yourself in a whole world of trouble. For example, if you write data to the primary a subsequent read may be routed to a secondary that has yet to have the data replicated to it.
This can be demonstrated by doing something like,

{% highlight javascript %}// Writes the object to the primary
db.collection.insert({_id: ObjectId("505bd76785ebb509fc183733"), key: "value"});
    
// This find is routed to a read-only secondary, and finds no results
db.collection.find({_id: ObjectId("505bd76785ebb509fc183733")});{% endhighlight %}
    
This is compounded if you have performance issues that cause the replication lag between a primary and its secondaries to increase to minutes or even hours in some cases. 

You can control whether a query is [run on secondaries](http://www.mongodb.org/display/DOCS/Querying#Querying-slaveOk%28QueryingSecondaries%29)
and also how many secondaries are replicated to [during the insert](http://docs.mongodb.org/manual/applications/replication/#replica-set-write-concern), but this will affect performance and could block forever in some cases!

## Range queries are indexed differently

I have found that range queries use indexes slightly differently to other queries. Ordinarily you would have the key used for sorting as the last element in a compound index. However, when using a range query like `$in` for example, Mongo applies the sort 
before it applies the range. This can cause the sort to be done on the documents in memory, which is pretty slow!

{% highlight javascript %}// This doesn't use the last element in a compound index to sort
db.collection.find({_id: {$in : [
  ObjectId("505bd76785ebb509fc183733"), 
  ObjectId("505bd76785ebb509fc183734"),
  ObjectId("505bd76785ebb509fc183735"),
  ObjectId("505bd76785ebb509fc183736")
]}}).sort({last_name: 1});{% endhighlight %}

At Heyzap we worked around the problem by building a caching layer for the query in Redis, but you can also run the same query twice if you only have two values in your `$in` statement or adjust your index if you have the RAM available.

You can read more about the [issue](http://blog.mongolab.com/2012/06/cardinal-ins/) or view a [ticket](https://jira.mongodb.org/browse/SERVER-3310).

## Mongo's BSON ID is awesome

Mongo's BSON ID provides you with a load of useful functionality, but when I first started using Mongo, I didn't realize half the things you can do with them. For example, the creation time of a BSON ID is stored in the ID. You can extract that time and you have a created_at field
for free! 

{% highlight javascript %}// Will return the time the ObjectId was created
ObjectId("505bd76785ebb509fc183733").getTimestamp();{% endhighlight %}

The BSON ID will also increment over time, so sorting by id will sort by creation date as well. The column is also indexed automatically, so these queries are super fast. You can read more about it on the [10gen site](http://www.mongodb.org/display/DOCS/Optimizing+Object+IDs).

## Index all the queries

When I first started using Mongo, I would sometimes run queries on an ad-hoc basis or from a cron job. I initially left those queries unindexed, as they weren't user facing and weren't run often. However this caused performance problems for other indexed queries, as the unindexed
queries do a lot of disk reads, which impacted the retrieval of any documents that weren't cached. I decided to make sure the queries are at least partially indexed to prevent things like this happening.

## Always run explain on new queries

This may seem obvious, and will certainly be familiar if you've come from a relational background, but it is equally important with Mongo. When adding a new query to an app, you should run the query on production data to check its speed. 
You can also ask Mongo to [explain](http://www.mongodb.org/display/DOCS/Explain) what its doing when running the query, so you can check things like which index its using etc.

{% highlight javascript %}db.collection.find(query).explain()
{
    // BasicCursor means no index used, BtreeCursor would mean this is an indexed query
    "cursor" : "BasicCursor",
    
    // The bounds of the index that were used, see how much of the index is being scanned
    "indexBounds" : [ ],
    
    // Number of documents or indexes scanned
    "nscanned" : 57594,
    
    // Number of documents scanned
    "nscannedObjects" : 57594,
    
    // The number of times the read/write lock was yielded
    "nYields" : 2 ,
    
    // Number of documents matched
    "n" : 3 ,
    
    // Duration in milliseconds
    "millis" : 108,
    
    // True if the results can be returned using only the index
    "indexOnly" : false,
    
    // If true, a multikey index was used
    "isMultiKey" : false
}{% endhighlight %}

I've seen code deployed with new queries that would take the site down because of a slow query that hadn't been checked on production data before deploy. It's relatively quick and easy to do so, so there is no real excuse not to!

##Profiler

MongoDB comes with a very useful [profiler](http://www.mongodb.org/display/DOCS/Database+Profiler). You can tune the profiler to only profile queries that take at least a certain amount of time depending on your needs. I like to have
it recording all queries that take over 100ms.

{% highlight javascript %}// Will profile all queries that take 100 ms
db.setProfilingLevel(1, 100);

// Will profile all queries
db.setProfilingLevel(2);

// Will disable the profiler
db.setProfilingLevel(0);{% endhighlight %}
    
The profiler saves all the profile data into the [capped collection](http://www.mongodb.org/display/DOCS/Capped+Collections) system.profile. This is just like any other collection so you can run some queries on it, for example
    
{% highlight javascript %}// Find the most recent profile entries
db.system.profile.find().sort({$natural:-1});
    
// Find all queries that took more than 5ms
db.system.profile.find( { millis : { $gt : 5 } } );
    
// Find only the slowest queries
db.system.profile.find().sort({millis:-1});{% endhighlight %}
    
You can also run the `show profile` helper to show some of the recent profiler output.

The profiler itself does add some [overhead](http://www.mongodb.org/display/DOCS/Database+Profiler#DatabaseProfiler-ProfilerOverhead) to each query, but in my opinion it is essential. Without it you are blind. I'd much rather add small overhead to the overall speed of the database
to give me visibility of which queries are causing problems. Without it you may just be blissfully unaware of how slow your queries actually are for a set of your users.

##Useful Mongo commands

Heres a summary of useful commands you can run inside the mongo shell to get an idea of how your server is acting. These can be scripted so you can pull out some values and chart or monitor them if you want.

- [db.currentOp()](http://www.mongodb.org/display/DOCS/Viewing+and+Terminating+Current+Operation) - shows you all currently running operations
- [db.killOp(opid)](http://www.mongodb.org/display/DOCS/Viewing+and+Terminating+Current+Operation) - lets you kill long running queries
- [db.serverStatus()](http://docs.mongodb.org/manual/reference/server-status-index/#server-status-example-instance-information) - shows you stats for the entire server, very useful for monitoring
- [db.stats()](http://docs.mongodb.org/manual/reference/database-statistics/) - shows you stats for the selected db
- [db.collection.stats()](http://docs.mongodb.org/manual/reference/collection-statistics/) - stats for the specified collection

##Monitoring

While monitoring production instances of Mongo over the last year or so, I've built up a list of key metrics that should be monitored.

### Index sizes

Seeing as how in MongoDB you really need your working set to fit in RAM, this is essential. At Heyzap for example, we would need our entire indexes to sit in memory, as we would quite often query our entire dataset when viewing older games or user profiles.

Charting the index size allowed Heyzap to accurately predict when we would need to scale the machine, drop an index or deal with growing index size in some other way. We would be able to predict to within a day or so when we would start to have problems
with the current growth of index.

### Current ops

Charting your current number of operations on your mongo database will show you when things start to back up. If you notice a spike in currentOps, you can go and look at your other metrics to see what caused the problem. Was there a slow query at that time? An increase in traffic? How can we 
mitigate this issue? When current ops spike, it quite often leads to replication lag if you are using a replica set, so getting on top of this is essential to preventing inconsistent reads across the replica set.

### Index misses

Index misses are when MongoDB has to hit the disk to load an index, which generally means your working set is starting to no longer fit in memory. Ideally, this value is 0. Depending on your usage it may not be. Loading an index from disk occasionally may
not adversely affect performance too much. You should be keeping this number as low as you can however.

### Replication lag

If you use replication as a means of backup, or if you read from those secondaries you should monitor your replication lag. Having a backup that is hours behind the primary node could be very damaging. Also reading from a secondary that is many hours behind the primary will likely cause your
users confusion.

### I/O performance

When you run your MongoDB instance in the cloud, using Amazon's EBS volumes for example, it's pretty useful to be able to see how the drives are doing. You can get random drops of I/O performance and you will need to correlate those with performance indicators such as the number of 
current ops to explain the spike in current ops. Monitoring something like [iostat](http://en.wikipedia.org/wiki/Iostat) will give you all the information you need to see whats going on with your disks.

### Monitoring commands

There are some pretty cool utilities that come with Mongo for monitoring your instances.

- [mongotop](http://docs.mongodb.org/manual/reference/mongotop/) - shows how much time was spend reading or writing each collection over the last second
- [mongostat](http://docs.mongodb.org/manual/reference/mongostat/) - brilliant live debug tool, gives a view on all your connected MongoDB instances

### Monitoring frontends

- [MMS](http://www.10gen.com/mongodb-monitoring-service) - 10gen's hosted mongo monitoring service. Good starting point.
- [Kibana](http://kibana.org/) - Logstash frontend. Trend analysis for Mongo logs. Pretty useful for good visibility.