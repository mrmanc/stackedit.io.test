---
layout: post
title:  "Data Modelling for MongoDB"
author: Alex Brown
tags: [MongoDB, Shippr]
---

This is the second of a two-part post looking at how your data model
affects MongoDB's performance. This post describes how we arrived at a
problematic data model, some of the changes we made to improve it and
describes what we learned along the way.

In the [first post]({{ site.github.url }}{% post_url
2017-05-01-investigating-query-performance-in-mongodb %}) we described
our app, Shippr, the performance issue we were having and the tools we
used to validate the behaviour of our MongoDB queries.

## How we Model Shippr's Data

When writing to MongoDB Shippr uses a library called
[Jongo](http://jongo.org/) to map POJOs into MongoDB documents for us.

So something like this

```java
public class Boat {
    public final String name;
    public final long sails;

    public Boat(String name, long sails) {
        this.name = name;
        this.sails = sails;
    }
}
```

```java
boats.insert(new Boat("Ship", 2L));
```

Becomes a document

```json
{
    "name" : "Ship",
    "sails" : 2
}
```

When we write data to MongoDB we first map our business logic objects
into these POJOs and vice versa when reading data back.

*[POJOs]: Plain Old Java Objects

## Reviewing the Data Model

We ended up at our current document structure since Jongo has
literally translated what's in our POJOs:

```java
public class ArrayInstances {
    public final Map<ArrayInstanceIdentifier, ArrayInstance> arrayInstances;
    /* SKIPPED */
}

public class ArrayInstance {
    public final Nodes nodes;
    public final ArrayInstanceIdentifier arrayInstanceIdentifier;
    public final ArrayLabel arrayLabel;
    /* SKIPPED */
}

public class Nodes {
    public final Map<NodeIdentifier, Node> nodes;
    /* SKIPPED */
}

public class Fleet {
    private final String _id;    
    private final String fleetName;
    private final String deploymentIdentifier;
    private final Map<String, ArrayInstance> arrayInstances;
    private final Set<String> nodeIdentifiers;
    /* SKIPPED */
}
```

Giving us:

```json
{
  "_id" : "d1e6328c-07a9-4a9d-adde-56e7543de0d1",
  "fleetName" : "dev_merlin-service-rest-search_388a72a740a495fbb92207377ddc7fea6b7dcef0.53",
  "deploymentIdentifier" : "9019bcb1-aa3a-4973-82ad-eb9ba57e0def",
  "arrayInstances" : {
    "e764e442-3746-4ed3-8e3e-20e53b687d00" : {
      "nodes" : {
        "b442ca37-93d1-4957-a49a-e3d81ebee35a" : {
          "arrayLabel" : "merlin-service-rest-search",
          "nodeIdentifier" : "b442ca37-93d1-4957-a49a-e3d81ebee35a",
          "nodeIpAddress" : "172.28.65.18",
          "nodeState" : "down"
        }
      },
      "arrayInstanceIdentifier" : "e764e442-3746-4ed3-8e3e-20e53b687d00",
      "arrayLabel" : "merlin-service-rest-search"
    }
  },
  "nodeIdentifiers" : [
    "b442ca37-93d1-4957-a49a-e3d81ebee35a"
  ]
}
```

I've skipped out a whole bunch of other fields, but you can see that
by this point we've already encountered an issue querying fleets by
node and have introduced some duplication to work around it,
`nodeIdentifiers` is a copy of all `nodeIdentifier` values on the
fleet. We'll see why this was necessary in a moment.

### Adding Indices

Okay, so let's have a go at trying to speed things up and add an index
to `nodeIdentifiers` and see if we get some improvement (the docs for
this are here
[method/db.collection.createIndex()](https://docs.mongodb.com/manual/reference/method/db.collection.createIndex/)).

```js
db.fleets.createIndex({ nodeIdentifiers : 1});
db.fleets.find({ nodeIdentifiers : "someNodeId" }).explain();
```

```json
{
    "winningPlan" : {
        "stage" : "FETCH",
        "inputStage" : {
            "stage" : "IXSCAN",
            "keyPattern" : {
                "nodes.nodeIdentifier" : 1
            },
            "indexName" : "nodes.nodeIdentifier_1",
            "isMultiKey" : true,
            "isUnique" : false,
            "isSparse" : false,
            "isPartial" : false,
            "indexVersion" : 1,
            "direction" : "forward",
            "indexBounds" : {
                "nodes.nodeIdentifier" : [ 
                    "[\"someNodeId\", \"someNodeId\"]"
                ]
            }
        }
    },
    "rejectedPlans" : []
}
```

A bit more of a response than last time, `IXSCAN` means 'index scan'
so this looks good, our query will now satisfy itself using the index
we built rather than checking every document. At this point in our
investigation we went on to re-run our performance tests and saw a
vast improvement in the response times for our node API endpoint.

But it still wasn't quite enough, array API requests were still too
slow so we wanted to create an index for that query as well.

However...

If you take a look at the argument to `createIndex` it takes a field
selector that you want to build your index from. If we look back at
the find-by-arrays query from the last post we see the *array
identifier is actually stored in the name of the field:*

```js
db.fleets.find({ arrayInstances.someArrayId : { $exists : true } });
```

Because the document field name isn't static we end up having to
create a *new* MongoDB query every time we want to look-up a fleet by
array.

This prevents us creating an index. It's also the reason we had to
duplicate node identifiers. If we want to find a fleet containing a
particular node we can't actually write a query that selects based on
node identifier alone, we also need to know the array identifier
before we can write the query.

So we've copied them all to a separate bit of the document so you
don't need to know the array to find the fleet containing a particular
node.

It's now unfortunately clear that we'll need to restructure the
document data model in order to support having indices for the queries
we want to do.

### Looking at Alternatives

After learning that having dynamic fields is a bad idea in
MongoDB-land we decided to push all the values into their
sub-documents and make our maps into arrays (we also took the
opportunity to rename `arrayInstances` to just `arrays`), so now
things look like this:

```json
{
 "_id" : "d1e6328c-07a9-4a9d-adde-56e7543de0d1",
 "fleetName" : "dev_merlin-service-rest-search_388a72a740a495fbb92207377ddc7fea6b7dcef0.53",
 "arrays" : [
    {
        "arrayIdentifier" : "e764e442-3746-4ed3-8e3e-20e53b687d00",
        "arrayLabel" : "merlin-service-rest-search",
        "nodes" : [
            {
                "nodeIdentifier" : "b442ca37-93d1-4957-a49a-e3d81ebee35a",
                "nodeIpAddress" : "172.28.65.18",
                "nodeState" : "pending"
            }
        ]
    }
 ]
}
```

That looks a bit more like it, no duplication, every document and
sub-document is just straight-forward collections of properties and
the relationships between values are captured by the structure of the
document.

Our queries are also looking nicer and are both indexable:

```js
db.fleets.find( { arrays.arrayIdentifier : "someArrayId" } );
```
```js
db.fleets.find( { arrays.nodes.nodeIdentifier : "someNodeId" } );
```

All good right? Unfortunately we're not quite there yet. We don't just
query our data, we also modify it. Most crucially we add nodes to
arrays and then update those nodes' state so Shippr can automate the
rest of the deployment and our users can get back to doing more
important things than wrangling servers.

So, adding a node:

```js
db.fleets.update(
    { arrays.arrayIdentifier : "someArrayId", arrays.nodes.nodeIdentifier : { $nin : [ "someNodeId" ] } },
    { $push : { arrays.$.nodes : { /* NODE JSON DOCUMENT */ } } }
);
```

This is complicated slightly by the fact that we might receive
multiple requests to add the same node at the same time, the query
part ensures we don't overwrite a node if it already exists. If
nothing matches the query part of an update, nothing gets written.

We then use the `$` or 'positional operator' in the update statement,
that says to use the element of the array that was matched in the
selection query (see more here about
[the $ operator](https://docs.mongodb.com/manual/reference/operator/update/positional/#up._S__)).

That works, so how about updating a node's state?

```js
db.fleets.update(
    { arrays.nodes.nodeIdentifier : "someNodeId"},
    { $set : { arrays.$.nodes.$.nodeState : "newState" } }
);
```

```
Too many positional (i.e. '$') elements found in path 'arrays.$.nodes.$'
```

Uh oh, MongoDB can only understand a single positional operator per
statement. This effectively means ***MongoDB cannot change the value
of documents in nested arrays*** without using dynamically generated
queries or otherwise knowing the first index as before.

For us this means that this document structure won't work for us either.

## Modelling Relationships

Checking the official MongoDB docs'
[Data Modeling Introduction](https://docs.mongodb.com/manual/core/data-modeling-introduction/),
they highlight that there are two main ways a relationship can be
represented in a mongo document;

A **fleet** having **arrays** each having **nodes** can be represented:

 * By document structure
 
   ```json
   {
     "_id" : "fleet",
     "arrays" : [
       {
         "_id" : "array",
         "nodes" : [
           {
             "_id" : "node"
           }
         ]
       }
     ]
   } 
   ```
  
 * By reference to another document's id

   ```json
   {
    "_id" : "fleet"
   }

   {
    "_id" : "array",
    "fleet_id" : "fleet"
   }

   {
    "_id" : "node",
    "array_id" : "array"
   }
   ```

Note how in the second option documents are storing both actual data
*and* structural data and that neither `fleet` nor `array` give any
indication they might have children (which could be fixed by having
bidirectional references, but then we've got *even more* metadata and
complexity in our model).

By compromising a bit and allowing an id reference back into our data
model we can flatten nodes to the point where they're both mutable and
indexable:

```json
{
  "arrays" : [
      {
        "arrayLabel": "merlin-service-rest-search",
        "arrayIdentifier": "e764e442-3746-4ed3-8e3e-20e53b687d00"
      }
    ],
  "nodes" : [
    {
      "arrayIdentifier" : "e764e442-3746-4ed3-8e3e-20e53b687d00",
      "nodeIdentifier" : "b442ca37-93d1-4957-a49a-e3d81ebee35a",
      "nodeIpAddress" : "172.28.65.18",
      "nodeState" : "down"
    }
  ]
}
```

This covers all our bases and allowed us to build indices to satisfy
all of our queries.

The resulting drop in query times ultimately meant our API was now
responsive enough to solve our original issue. Success!

## In Summary

So when it comes to designing your own MongoDB document structures
here are some of the things to bear in mind:

* **Avoid dynamic field names**

  It's difficult for MongoDB to address them, and they cannot be added
  to indices. Treat field names as part of your model's structure,
  rather than as values.

* **Prefer arrays of documents over maps**

  Map-like structures result in documents which contain only dynamic fields.
  
  You should try to figure out how to represent your maps in your
  document's structure, even an array of sub-documents like this is
  better than nothing:

  ```json
  {
      "_id" : "key",
      "value" : { /* Your sub-document */ }
  }
  ```

* **If your sub-document needs to be mutated or indexed, it needs to
  be flattened as much as possible**

  Either all the way to its own collection or as an array at the root
  of its parent document. Avoid using references to other ids unless
  you absolutely need to in order to keep your documents simple.
  
  Be mindful of the concurrency guarantees your application needs when
  doing this, atomicity is only guaranteed to changes made to a single
  document, so separating things to multiple collections and writing
  to them all at once will not be atomic. There's more detail in the
  MongoDB docs here:
  [Atomicity and Transactions](https://docs.mongodb.com/manual/core/write-operations-atomicity/).

