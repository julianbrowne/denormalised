
#DENORMALISED - 20th September 2012

##Deep dive into MongoDB

MongoDB has found a sweet spot in enterprise adoption due to its ease of use, powerful query model and natural progression for those coming from a SQL mindset. But under the hood it has some unusual design approaches that can trip up the unwary.  

Equal Experts has first hand experience of putting MongoDB to use on a number of production systems. In this talk Julian Browne will talk about the power and the pitfalls of MongoDB using practical examples that highlight its inner workings.  

##Prequisites

Assumes MongoDB has been downloaded and that there's a writable /data/db directory for mongod to access

##Script

###Basics

cd to mongodb bin directory and start it up

	$ mongod --rest

The --rest option provides a useful http connection at http://localhost:28017/

Let's see what mongod is up to:

	$ mongostat  

Important columns are query/update/delete/getmore (what clients are up to); mapped (size of data, starts at 0m); the process resource utilisation (vsize, res, faults) and the frequency of the global lock (locked)

###Exploring MongoDB

Let's make some data using the mongo shell

	$ mongo
	> 

Help is your best friend

	> help
	db.help()                    help on db methods
	db.mycoll.help()             help on collection methods
	sh.help()                    sharding helpers
	rs.help()                    replica set helpers
	help admin                   administrative help
	help connect                 connecting to a db help
	help keys                    key shortcuts
	help misc                    misc things to know
	help mr                      mapreduce

	show dbs                     show database names
	show collections             show collections in current database
	show users                   show users in current database
	show profile                 show most recent system.profile entries with time >= 1ms
	show logs                    show the accessible logger names
	show log [name]              prints out the last segment of log in memory, 'global' is default
	use <db_name>                set current database
	db.foo.find()                list objects in collection foo
	db.foo.find( { a : 1 } )     list objects in foo where a == 1
	it                           result of the last line evaluated; use to further iterate
	DBQuery.shellBatchSize = x   set default number of items to display on shell
	exit                         quit the mongo shell

Also, for demos etc, cls to clear the screen and start at the top is quite useful:
 
	> cls

Another useful feature is that commands minus their brackets explain what they do:

	> db.stats
	function (scale) {
    	return this.runCommand({dbstats:1, scale:scale});
	}

What databases exist?

	> shows dbs

There are no databases yet, so let's make one:

	> use dave

"db" is conventional notation for 'the current' database. Now let's create a collection called "colin" in dave:

	> db.createCollection('colin')

This make a conventional, empty, collection to put documents in. And we can see the affect it's had by running mongostat in the MongoDB bin directory

	$ mongostat
	
This now shows some storage as 'mapped'. If we go the file system in /data/db we can see the files that have been created to manage out data:

	$ cd /data/db
	$ ls -lsk

We should have a db storage file of 64Mb (dave.0) plus a namespace file (dave.ns), plus a 128Mb file created for future use to save time later (dave.1). This ahead of time creation can be prevented if desired with a --noprealloc option to mongod  

Now let's add a simple document from the mongo shell

	> db.colin.save({hello: "world"})

And back to the file system to check out what's happened. We can look inside the dave.0 file and actually see our JSON doc saved in BSON form:

	$ od -c dave.0

Note: field names are stored as is (not compressed or tokenised) and repeated for every document so it's important to keep key names short to minimise storage needs and hence allow for more docs to be in memory concurrently.

###Populating MongoDB

First, let's add some useful functions to the shell to make adding some interesting documents easier:

	> // generate random string of length
	
	function randString(length) {
		var text  = "";
		var chars = "abcdefghijklmnopqrstuvwxyz0123456789";
    	for(var i=0; i < length; i++ )
    		text += chars.charAt(Math.floor(Math.random() * chars.length));
    	return text;
	}

	> // generate random number up to max

	function randInt(max) {
		return Math.floor(Math.random() * max);
	}

	// generate a random number of likes

	function randLikes(max) {
		var sports = [ "football", "rugby", "hockey", "tennis", "running", "equestrian", "shotput", "javelin", "high jump", "long jump" ];
		var likes = [ ];
		for(var i=0; i < max; i++) {
			var likeId = randInt(sports.length);
			likes.push(sports[likeId]);
			sports = sports.slice(0,likeId).concat(sports.slice(likeId+1));
		}
		return likes;
	}

Now we can add half a million documents to the colin collection:

	>	for(var id=0; id < 500000; id++) {
			db.colin.save({
				"_id"   : id,
				"name"  : randString(20),
				"age"   : randInt(100),
				"likes" : randLikes(3)
			});	
		}

And mongostat provides some useful information about how things are performing

	$ mongostat

A Macbook Pro with SSD typically creates these document at a rate of between 9,000 and 12,000 per second. Note the mapped data total rising and the global lock being invoked. Writes are of course async so this is a useful baseline figure, not a meaningful benchmark.

Back in the shell we can see what MongoDB thinks we have:

	> db.colin.stats()
	

###Finding Data

To find all docs in the collection we pass an empty JSON document to colin's find function

	> db.colin.find({})

We can also page through data by assigning those results to a cursor

	> var c = db.colin.find({})  
	> a.hasNext()					// more data?
	true
	> a.next()  					// get next document

Get total number documents in the collection

	> db.colin.find({}).count() 
	
Or, simpler:

	> db.colin.count()  

Find has a lot of options, one nice one is regex support:

	> db.colin.find({name: /^Dave/})

Though by default searches are case sensitive by default so better to try:

	> db.colin.find({name: /^Dave/i})

Want to find all people who like football? Can search into arrays to without extra command overhead:

	> db.colin.find({"likes" : "football"})

Find all people who are 18

	> db.colin.find({"age" : 18})

Find how many

	> db.colin.find({"age" : 18}).count()  


Find all people who are under 18

	> db.colin.find({"age" : { $lt : 18 } })  

How many are there:

	> db.colin.find({"age" : { $lt : 18 } }).count()

Note that:

	> db.colin.find({"age" : 18})

takes a perceptible amount of time to return with 500,000 documents. When developing for a production system this is a warning sign worth investigating. First step is to tell the profiler to report on long running queries. In this case let's set the reporting level very low - to 20ms

	> db.setProfilingLevel(1,20)

Now when those queries are run again we'll get a useful report logged to the console where mongod was run  

It's also worth using explain() to find out how MongoDB is servicing the query:

	> db.colin.find({"age" : 18}).explain()

Explain tells us all 500,000 documents are being scanned for that query. If this query is important then we should add an index to help MongoDB out:

	> db.colin.ensureIndex({"age" : 1})
	
Checking explain again. Should show an enormous drop in response time (from 140ms to about 5ms) and documents scanned. Note though that indexes aren't free. They take up space and they also need to be updated when new documents are added. Index use must be balanced against against write impact.  

Indexes can also enforce uniqueness for fields.  

What about that count operation?

	> db.colin.find({age: 25}).count()

MongoDB doesn't use counting btrees for its indexes so even with an index count() queries on large document sets can still be slower than expected.

If after creating an index you would like to see how bad performance would have been without it, you can hint the query to ignore the index:

	> db.colin.find({age: 25}).hint({$natural:1})

The aggregation function provides a useful alternative for queries that may in the past have required map/reduce

	> db.colin.aggregate( { $match : { "likes" : "football" } } );  

This will fail with 500,000 documents as output must remain within 16Mb doc size limit. Here's one that narrows the result set and should work:

	> db.colin.aggregate( { $match : { "likes" : "football", age : { $lt : 20 } }  } );  

###Updating data

Changing one field for a field of identical (or smaller) length is very quick in MongoDB:

	> for(var id=0; id < 500000; id++) {
			db.colin.update({"_id" : id}, { $set : { "postcode" : "EC1V 7DP" } });
		}

Transaction rates (async) can reach 45,000/second if there's no index on colin and around 11,000/second with an index

	> db.colin.dropIndexes()

Will remove all non _id indexes for this test. Checking

	> db.colin.stats()

Shows what MongoDB is using for the Padding Factor between documents. This is intelligently increased when documents grow a lot.

Transaction rates can be seriously impacted if a document grows beyond it's current slot and has to be moved to another place in the data file. This script makes the 20 char name 50 chars long:

	> for(var id=0; id < 500000; id++) {
			var newName = randString(50);
			db.colin.update({ "_id" : id }, { $set : { "name" : newName } } );
		}

and performs at about 15,000 writes/second without an index. 10,000 writes/sec with an index on age and 3,000 writes.sec with an index on name. Worth comparing the 50,000 writes/sec when doc stays the same size or shrinks to 3,000 writes/sec when there's an index on name and name is growing in size. In some cases it's worth padding out documents when they are created to keep expansion to a minimum.

Collections can become fragmented, just like a file system, over time.

	> db.runCommand( { compact : 'colin' } )

will defrag them, but note the % global lock in mongostat when this is happening. Btest not to do this in a live production environment when there's a lot of documents.

###Deleting Data

	> db.colin.drop()

This empties out the data but db files remain

	$ cd /data/db  
	$ ls -lsk

But:

	> db.dropDatabase()

Removes the files

##Further Reading

- Additional javascript examples can be found in the test suite at https://github.com/mongodb/mongo/tree/master/jstests/
- Always check out the JIRA projects at https://jira.mongodb.org/secure/Dashboard.jspa as this helps distinguish between genuine bugs and intentional but non-obvious behaviour.


