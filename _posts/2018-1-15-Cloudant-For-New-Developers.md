---
layout: post
title: Guidelines for how to use Cloudant effectively as a developer
---

![_config.yml]({{ site.baseurl }}/images/config.png)

_Audience: software developers set to use Cloudant in a project, but with little
Cloudant-specific experience._

The idea is to lay out a set of guidelines that if followed leads to the [pit of success](https://blog.codinghorror.com/falling-into-the-pit-of-success/) -- where
your application will be using Cloudant "following the grain". These guidelines will necessarily be
stricter than they need to be. As you grow in experience, you'll learn which such guidelines can be
relaxed.

Some of this is subject to [Wittgenstein's ladder](https://en.wikipedia.org/wiki/Wittgenstein%27s_ladder) --
that is to say that some explanations are simplified or incomplete in order to get concepts across.
This is very much deliberate. 

## Rule number 0: Understand the API you are targeting

You may use [Java](https://github.com/cloudant/java-cloudant), or 
[Node](https://github.com/cloudant/nodejs-cloudant) or some other use-case specific language or platform 
that likely comes with convenient client-side libraries that integrate Cloudant access nicely, following 
the conventions you expect for your tools. This is great for programmer efficiency, but also hides the API from view.

This is what you want, of course. However, understanding the underlying API is _vital_ when it comes 
to troubleshooting, and when reporting problems. When you report a suspected problem to Cloudant,
you need to provide a way for us to reproduce the problem. This does emphatically _not_ mean cutting and pasting
a hefty chunk of your application's Java source into a support ticket, as we're most likely not able 
to build it, and introduces uncertainties as to where the problem may be -- your side or our side?

Cloudant's support teams will usually ask you to provide the set of API calls, ideally as a set of [curl](https://curl.haxx.se/) 
commands that they can run, that demonstrates the issue. Adopting this approach to troubleshooting
as a general rule also makes it easier for you to pinpoint where issues are failing. If your code
is behaving unexpectedly, try to reproduce the problem using only direct access to the API. If you
can't, the problem isn't with the Cloudant service itself. 

If you suspect that a problem you've encountered lies with an officially supported client library,
then try to construct a small, self-contained code example that demonstrates the issue, with as few 
other dependencies as possible. If you're using Java it is helpful to us if you can use a minimal [test harness](https://github.com/mikerhodes/java-cloudant-minimal)
to highlight library issues.

It's common for Cloudant to receive support tickets that state that "Cloudant is broken because my application is 
slow" without much in terms of supporting evidence. Nearly always this is traced back to issues in 
the application code on the client side, or misconceptions about how Cloudant works. Not always, but _nearly_ always.

By understanding the API better, you also gain experience in how Cloudant behaves, especially in terms
of performance.

For the duration of this, we will only use direct access to the API for this reason. This will feel
clunky if you're not used to working this way, but it will pay you back many times over time.

> Cloudant API [docs](https://console.bluemix.net/docs/services/Cloudant/api/index.html#api-reference-overview).

## Rule number 1: Documents should group data that mostly change together

When you start to model your data, sooner or later you'll run into the issue of how your documents
should be structured. You've gleaned that Cloudant doesn't enforce any kind of normalisation and that
it has no transactions of the type you're used to from say [Postgres](https://www.postgresql.org/), 
so the temptation can be to cram as much as possible into each document, given that this would also 
save on HTTP overhead.

This is a bad idea, in the main.

If your model groups information together that doesn't change together, you're more likely to suffer 
from update conflicts.

Consider a situation where you have users, each having a set of orders associated with them. One way
might be to represent the orders as an array in the user document:

    { // DON'T DO THIS
        "customer_id": 65522389,
        "orders": [
            {
                "order_id": 887865,
                "items": [
                    {
                        "item_id": 9982,
                        "item_name": "Iron sprocket",
                        "cost": 53.0
                    },
                    {
                        "item_id": 2932,
                        "item_name": "Rubber wedge",
                        "cost": 3.0
                    }
                 ]
            }    
        ]
    }

To add a new order, I need to fetch the complete document, unmarshal the JSON, add the item, 
marshal the new JSON, and send it back as an update. If I'm the only one doing so, it may work for a 
while. If the document is being updated concurrently, or being replicated, we'll likely see update 
conflicts.

Instead, keep orders separate as their own document type, referencing the customer id. Now, the model
is immutable. To add a new order, I simply create a _new_ order document in the database, which cannot
generate conflicts. 

To be able to retrieve all orders for a given customer, we can employ a _view_, which we'll cover later.

Avoid constructs that rely on updates to parts of existing documents, where possible. Bad data models are
often extremely hard to change once you're in production.

> Cloudant guide to [data modelling](https://console.bluemix.net/docs/services/Cloudant/guides/model_data.html#my-top-5-tips-for-modelling-your-data-to-scale).

## Rule number 2: Keep documents small

Cloudant imposes a max doc size of 1M. This does not mean that a close-to-1M document size is a good idea.
On the contrary, if you find you are creating documents that exceed single-digit kB, you should probably 
revisit your model. A number of things in Cloudant becomes less performant as documents grow. JSON decoding
is costly, for example.

## Rule number 3: Avoid using attachments

The only real reason why Cloudant/CouchDB support attachments at all is backwards compatibility with
so-called "couchapps" -- an attempt to make CouchDB a simple webapp server. No one in their right mind
uses couchapps today, but the ability to store binary blobs (images etc) remain, and for some reason
some developers are drawn to this capability.

For a number of reasons, this is a bad idea:

1. Cloudant is expensive as a block store at $1/G
1. Cloudant's internal implementation is not geared to handle efficiently large amount of binary data

So: slow and expensive, basically. Not what you want.

If you need to store binary data alongside Cloudant documents, use a separate solution that is more
suited for this purpose, and store only the attachment _metadata_ in the Cloudant document. Yes, that means some extra code
you need to write to upload the attachment to an object store, verify that it's succeeded before storing
the token or URL to the attachment in the Cloudant document.

Your databases will be smaller, cheaper, faster, and easier to replicate.

> Cloudant docs on [attachments](https://console.bluemix.net/docs/services/Cloudant/api/attachments.html#attachments).

> Detatching Cloudant attachments to [Object Store](https://medium.com/ibm-watson-data-lab/detaching-cloudant-attachments-to-object-storage-with-serverless-functions-99b8c3c77925).

## Rule number 4: Fewer databases are better than many

If you can, try to limit the number of databases per Cloudant account to 500 or fewer. Whilst Cloudant
can safely handle more, there are several use cases that are adversely affected by large numbers of 
databases in an account.

The replicator scheduler has a limited number of simultaneous replication jobs it is prepared to run.
That means that as the number of databases grow, the replication latency is likely to increase if you 
try to replicate everything contained in an account.

There is an operational aspect which is the flipside of the same coin: Cloudant itself relies on 
replication, too, in order to move accounts around. By keeping down the number of databases you
help us help you should you want to shift your account from one location to another.

So when should you use a single database and distinguish between different document types using 
views, and when should you use multiple databases to model your data? Cloudant can't federate views
across multiple databases so if you have data that is unrelated to the extent that they will 
never be "joined" or queried together, then that data could be a candidate for splitting across
multiple databases.

## Rule number 5: Avoid the "database per user" anti-pattern like the plague

If you're building out a multi-user service on top of Cloudant it is tempting to let each user 
store their data in a separate database under the application account. That works well, mostly,
as the number of users is small. 

Now add the need to derive cross-user analytics. The way you do that is to replicate all the user
databases into a single analytics DB. All good. Now, this app suddenly becomes successful, and the
number of users grow from 150 to 20,000. Now we have 20,000 replications just to keep the analytics
DB current. If we also want to run in an active-active DR setup, we add another 20,000 replications
and basically the system will stop functioning. 

Instead, multiplex user data into fewer databases, or shard users into a set of databases or 
accounts, or both. That way there is no need to replicate to provide an analytics DB, but auth
becomes more complicated as Cloudant only provides authentication at database level. 

It's worth stating that the "database-per-user" approach is tempting because Cloudant permissions 
are "per database" -- it's not really the users' fault that this pattern has emerged.

## Rule number 6: Avoid writing custom JavaScript reduce functions

The map-reduce views in Cloudant are awesome. However, with great power comes great responsibility.
The map-part of a map-reduce view is built incrementally, so shoddy code in the map impacts only
indexing time, not query time. The reduce-part, unfortunately, will execute at query time. Cloudant
provides a set of built-in reduce functions that are implemented internally in
[Erlang](https://www.erlang.org/), and not running as JavaScript. 

If you find yourself writing reduce functions, stop and consider if you could re-organise your data
so that this isn't necessary, or you're able to rely on the built-in reducers.

Actually, we can make that a bit more hardline: if you find you are using custom reducers, you're doing it wrong.

> Cloudant docs on [reducers](https://console.bluemix.net/docs/services/Cloudant/api/creating_views.html#reduce-functions)

## Rule number 7: Understand the trade-offs in emitting data or not into a view

As the document referenced by a view is always available using `include_docs=true`, it is 
possible to do something like 

    emit(doc.indexed_field, null);
    
in order to allow lookups on `indexed_field`. This has advantages and disadvantages.

1. The index is compact -- this is good, as index size contributes to storage costs
1. The index is robust -- as the index does not store the document, you can access any field without thinking ahead of what to store in the index

The disadvantage though is that getting the document back is more costly than
the alternative of emitting data into the index itself, as the database first 
has to look up the requested key in the index, and then read the associated document. Also, 
if you're reading the whole document, but actually need only a single field, you're making the
database read and transmit data you don't need.

This also means that there is a potential race here -- the document may have changed, or
been deleted between the index and document read (although unlikely in practice).

Emitting data into the index (a so-called projection) means that you can fine-tune the exact
subset of the document that you actually need. In other words, you don't need to emit the whole 
document. Only emit a value which represents the data you need in the app i.e. a cut-down object 
with minimal details, for example: 

    emit(doc.indexed_field, {name: doc.name, dob: doc.dob});

Of course, if you change your mind what fields you want to emit, the index will need
rebuilding.

Cloudant Query's (CQ) JSON index uses views this way under the hood. CQ can be a convenient
replacement for some types of view queries, but not all. Do take the time to understand
when to use one or the other.

> Cloudant guide to using [views](https://console.bluemix.net/docs/services/Cloudant/api/using_views.html#using-views).

> Performance implications of using [include_docs](https://console.bluemix.net/docs/services/Cloudant/api/using_views.html#include_docs_caveat).

## Rule number 8: Never rely on the default behaviour of Cloudant Query's no-indexing

It's tempting to rely on CQs ability to query without creating explicit indexes. This
is extremely costly in terms of performance, as every lookup is a full scan of the database
rather than an indexed lookup. If your data is small, this won't matter, but as the dataset
grows, this will become a problem for you, and for the cluster as a whole. It is likely
that we will limit this facility in the near future. The Cloudant dashboard allows you
to create indexes in an easy way.

Creating indexes and crafting CQs that take advantage of them requires some flair. To identify 
which index is being used by a particular query, send a POST to the `_explain` endpoint for 
the database, with the query as data. 

> Cloudant Query [docs](https://console.bluemix.net/docs/services/Cloudant/api/cloudant_query.html#query).

## Rule number 9: In Cloudant Search (or CQ indexes of type `text`), limit the number of fields

Cloudant Search and CQ indexes of type `text` (both of which are 
[Apache Lucene](https://lucene.apache.org/core/) under the hood) allows you to index any number of 
fields into the index. We've seen some examples where this is abused either deliberately, or most 
often fat-fingered. Plan your indexing to comprise only the fields required by your actual queries. 
Indexes take space and can be costly to rebuild if the number of indexed fields are large.

There's also the issue of which fields you store in a Cloudant Search. Stored fields are 
retrieved in the query without doing `include_docs=true` so the trade-off is similar to Rule 7.

> Cloudant Search [docs](https://console.bluemix.net/docs/services/Cloudant/api/search.html#search).

## Rule number 10: Resolve your conflicts

Cloudant is designed to treat conflicts as a natural state of data in a distributed system.
This is a powerful feature that helps a Cloudant cluster be available at all times. However,
the assumption is that conflicts are still reasonably rare. Keeping track of conflicts 
in Cloudant's core has significant cost associated with it.

It is perfectly possible to just ignore conflicts, and the 
database will carry on operating by choosing a random, but deterministic revision of conflicted
documents. However, as the number of unresolved conflicts grow, the performance of the database
goes down a black hole, especially when replicating. As a developer, it's your responsibility
to check for, and to resolve conflicts -- or even better, employ data models that makes conflicts
impossible.

If you routinely create conflicts, you should consider model changes.

> Cloudant guide to [conflicts](https://console.bluemix.net/docs/services/Cloudant/guides/conflicts.html#conflicts).

> Cloudant guide to versions and [MVCC](https://console.bluemix.net/docs/services/Cloudant/guides/mvcc.html#document-versioning-and-mvcc).

> Three-part blog series on [conflicts](https://developer.ibm.com/dwblog/2015/cloudant-document-conflicts-one/).

## Rule number 11: Don't delete documents

Deleting a document doesn't actually purge a document from a Cloudant database. 
Deletion is implemented by writing a new revision of a the document under deletion, with an added
field `_deleted: true`. This special revision is called a `tombstone`. Tombstones still take up space
and are passed around by the replicator, too.

Models that rely on frequent deletions of documents are not suitable for Cloudant.

## Rule number 12: Be careful with updates

It is more expensive in the longer run to mutate existing documents than to create new ones, as
Cloudant will always need to keep the document tree _structure_ around, even if internal nodes
in the tree will be stripped of their payloads. If you find that you create long revision trees, 
your replication performance will go down. Moreover, if your update frequency goes above say once 
or twice every few seconds, you're more likely to produce update conflicts. 

Prefer models that are immutable.

The obvious question after rules 11 and 12 is -- won't the data set grow unbounded if my model is
immutable? If you accept that deletes don't purge the deleted data and that updates are actually
not updating in place, in terms of data volume growth there is not much difference. Managing 
data volume over time requires different techniques. The only way to truly reclaim space is to
delete _databases_, rather than documents. You can replicate only winning revisions to a new
database and delete the old to get rid of lingering deletes and conflicts. Or perhaps you can
build it into your model to regularly start new databases (say 'annual data') and archive off
(or remove) outdated data, if your use case allows.

## Rule number 13: Replication isn't magic

"So let's set up three clusters across the world, Dallas, London, Sydney, with bidirectional
synchronisation between them to provide real time collaboration between our 100,000 clients."

No.

Cloudant is good at replication. It's so effortless that it can seem like magic, but note that
it makes no latency guarantees. In fact, the whole system is designed with eventual consistency
in mind. Treating Cloudant's replication as a real time messaging system will not end up in
a happy place. For this use case, put a system in between that was designed for this purpose, 
such as [Apache Kafka](https://kafka.apache.org/).

It's difficult to put a number on replication throughput -- the answer is always "it depends".
Things that impact replication performance include, but are not limited to:

1. Change frequency
1. Document size
1. Number of simultaneous replication jobs on the cluster as a whole
1. Wide (conflicted) document trees

> [Blog post](https://dx13.co.uk/articles/2017/11/7/cloudant-replication-topologies-and-failover.html) on 
replication topology.

> Cloudant guide to [replication](https://console.bluemix.net/docs/services/Cloudant/guides/replication_guide.html#replication).

> Updates to the replication [scheduler](https://developer.ibm.com/dwblog/2017/replicator-apache-couchdb-cloudant/).

## Rule number 14: Use the bulk API

Cloudant has nice API endpoints for bulk loading (and reading) many documents at once. This can be much more 
efficient than reading/writing many documents one at a time. The write endpoint is `${database}/_bulk_docs`. Its
main purpose is to be a central part in the replicator algorithm, but it's available for your use, too.

Its more recently added cousin -- `_bulk_get` -- is perhaps less useful, as you can already read many documents
at once by issuing a `POST` to `_all_docs`. `_bulk_get` is much slower than `_all_docs` because it is fetching all 
revisions. Probably not for the faint-hearted, unless you're making your own replicator.

Note that with `_bulk_docs`, in addition to creating docs you can also update and delete. Some client libraries,
including [PouchDB](https://pouchdb.com/), implements create, update and delete even for single documents this 
way for fewer code paths. 

Here is an example creating one new, updating an existing and deleting a third document:

    curl -XPOST 'https://ACCT.cloudant.com/DB/_bulk_docs' \
        -H "Content-Type: application/json" \
        -d '{"docs":[{"baz":"boo"}, \
                     {"_id":"463bd...","foo":"bar"},  \
                     {"_id":"ae52d...","_rev":"1-8147...","_deleted": true}]}'
                     
To fetch a fixed set of docs using `_all_docs`, POST with a `keys` body:

    curl -XPOST 'https://ACCT.cloudant.com/DB/_all_docs' \
        -H "Content-Type: application/json" \
        -d '{"keys":["ab234....","87addef...","76ccad..."]}'

> Note that Cloudant (at the time of writing) imposes a max request size of 1M, so bulk
requests exceeding this will be rejected.

> Cloudant bulk operations [docs](https://console.bluemix.net/docs/services/Cloudant/api/document.html#bulk-operations).

## Rule number 15: Eventual Consistency is a harsh taskmaster (a.k.a don't read your writes)

Eventual consistency is a great idea on paper, and a key contributor to Cloudant's ability to scale out in practice.
However, it's fair to say that the mindset required to develop against an eventually consistent
data store does not feel natural to most people. 

You often get stung when writing tests:

1. Create a database
1. Populate the database with some test data
1. Query the database for some subset of this test data
1. Verify that the data you got back is the data you expected to get back

Nothing wrong with that? That works on every other database you've ever used, right? Not on 
Cloudant. Or rather, it works 99 times out of 100.

The reason for this is that there is a (mostly) small inconsistency window between writing
data to the database, and this being available on all nodes of the cluster. As all nodes
in a cluster are equal in stature, there is no guarantee that a write and a subsequent read
will be serviced by the same node, so in some circumstances the read may be hitting a node
before the written data has made it to this node.

So why don't you just put a short delay in your test between the write and the read? That
will make the test less likely to fail, but the problem is still there.

Cloudant has no transactional guarantees, and whilst document writes are atomic (you're 
guaranteed that a document can either be read in its entirety, or not at all), there is
_no way_ to close the inconsistency window (except for the trick below). It's there by design. 

A more serious concern that should be at the forefront of every developer's mind is that
you can't safely assume that data you write will be available to anyone else at a specific
point in time. This takes some getting used to if you come from a different kind of database
tradition.

> Testing Tip: what you _can_ do to avoid the inconsistency window in testing is to test against 
an single-node instance of Cloudant or CouchDB running say in Docker (docker stuff 
[here](https://hub.docker.com/_/couchdb/)). A single node removes the eventual consistency issue, 
but beware that you are then testing against an environment that behaves differently to what you
will target in production. _Caveat Emptor_.

## Rule number 16: Don't mess with Q, R and N unless you really know what you are doing

Cloudant's quorum and sharding parameters, once you discover them, seem like tempting options
to change the behaviour of the database. Stronger consistency -- surely just set the write
quorum to the replica count?

No! Recall that there is _no way_ to close the inconsistency window in a cluster.

Don't ever go there, at least not until you've consulted Cloudant on your particular situation.

There are times when tweaking the shard count is essential to squeeze performance out of 
a cluster, but if you can't say why this is, you're likely to make your situation worse. 

## Rule number 17: Design document (ddoc) management requires some flair

As your data set grows, and your number of views goes up, sooner or later you will want to ponder
how you organise your views across ddocs. A single ddoc can be used to form a 
so-called `view group`, a set of views that belong together by some metric that makes sense
for your use case. If your views are pretty static that makes your view query URLs semantically
similar for related queries. It's also more performant at index time because the index loads the 
document once and generates multiple indexes from it.

Ddocs themselves are read and written using the same read/write endpoints as any other document.
This means that you can create, inspect, modify and delete ddocs from within your application.
However, even small changes to ddocs can have big effects on your database. When you update a 
design document, _all_ views in the ddoc become unavailable until indexing is complete. This can
be problematic in production. To avoid it you have to do a crazy  design-document-swapping 
dance (see [couchmigrate](https://github.com/glynnbird/couchmigrate)).

In most cases, this is probably not what you want to have to deal with. As you start out it is 
most likely safer to have a one-view-per-ddoc policy. 

Also, in case it isn't obvious, views are _code_ and should be subject to the same processes you
use in terms of source code version management for the rest of your application code. How to 
achieve this may not be immediately obvious. You could version the JS snippets and then cut & paste
the code into the Cloudant dashboard to deploy whenever there is a change, and yes, we all resort
to this from time to time. However, there are much better ways, and this is the one remaining 
reason to use some of the concepts surrounding the `couchapp` concept that was briefly touched upon
earlier. Several couchapp tools exist that are there to make the deployment of a couchapp -- 
including its views, crucially -- easier.

Using a couchapp tool means that you can automate deployment of views as needed. 

> See for example [couchapp](https://github.com/couchapp/couchapp) and [situp](https://github.com/drsm79/situp).

> Cloudant guide to design doc [management](https://console.bluemix.net/docs/services/Cloudant/guides/design_document_management.html#design-document-management).

## Rule number 18: Cloudant is rate limited -- let this inform your code

Cloudant-the-service, unlike vanilla CouchDB, is sold on a "reserved throughput capacity" model. 
That means that you pay for the right to use up to a certain throughput, rather than the throughput you 
actually end up consuming. This takes a while to sink in for most new users. One somewhat flaky comparison 
might be that of a cell phone contract where you pay for a set number of minutes regardless of whether 
you end up using them or not.

Although the cell phone contract comparison doesn't really capture the whole situation. There is no constraint
on the sum of requests you can make to Cloudant in a month, the constraint is on how _fast_ you make requests.

It's really a promise that you make to Cloudant, not one that Cloudant makes to you: you promise to not 
make more requests per second than what you said you would up front. A top speed limit, if you like. If you
transgress, Cloudant will fail your requests with a status of `429: Too Many Requests`. It's your responsibility
to look out for this, and deal with it appropriately, which can be difficult when you've got multiple app servers. 
How can they coordinate to ensure that they collectively stay below the requests-per-second limit? 

Cloudant's official client libraries have some built-in provision for this that can be enabled (note: off by default
to force you to think about this), following a 
"back-off & retry" strategy. However, if you rely on this facility alone you will eventually be disappointed.
Back-off & retry only helps in cases of temporary transgression, not a persistent butting up against your 
provisioned throughput capacity. 

Your business logic _must_ be able to handle this condition. Another way to look at it is that you get the 
allocation you pay for. If that allocation isn't sufficient, the only solution is to pay for a higher allocation.

Provisioned throughput capacity is split into three different buckets: _Lookups_, _Writes_
and _Queries_. A _Lookup_ is a "primary key" read -- fetching a document based on its `_id`. A _Write_ is storing a 
document or attachment on disk, and a _Query_ is looking up documents via a secondary index (any API endpoint that
has a `_design` or `_find` in it).

You get different allocations of each and the ratios between them are fixed. This fact can be used
to optimise for cost. As you get 20 _Lookups_ for every 1 _Query_ (per second), if you find that you're 
mainly hitting the _Query_ limit but you have plenty headroom in _Lookups_, it may be possible to reduce
the reliance on _Queries_ through some remodelling of the data or perhaps doing more work client side.

The corollary here though is that you can't assume that any 3rd party library or framework will 
optimise for cost ahead of convenience. Client-side frameworks that support multiple persistence layers
via plugins are unlikely to be aware of this, or perhaps even incapable of make such trade-offs. 

This is worth checking before committing to a particular tool.

It is also worth understanding that the rates aren't directly equivalent to HTTP API endpoint calls. You should
expect that for example a bulk update will count according to its constituent document writes.

> Cloudant docs on [plans and pricing](https://console.bluemix.net/docs/services/Cloudant/offerings/bluemix.html#ibm-cloud-public) on IBM Public Cloud.

## Thanks

Thanks to Glynn Bird, Robert Newson and Rich Ellis for helpful comments. Any remaining errors are mine alone, and nothing to do with them.