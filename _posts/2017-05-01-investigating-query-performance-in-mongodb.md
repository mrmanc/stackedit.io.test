---
title: Investigating Query Performance in MongoDB
date: 2017-05-01 00:00:00 +01:00
tags:
- MongoDB
- Shippr
layout: post
author: Alex Brown
---

This is the first of a two-part post looking at how your data model
affects MongoDB's performance. This post describes a performance issue
with one of our applications and how we came to the decision that our
document model was ultimately the cause.

The second post will then describe how we arrived at this poorly
performing document model, what changes we made to improve things and
to summarise what we learned.

## Introducing Shippr

Shippr is the name of the orchestration component of our internal
cloud platform and is one of the tools enabling
[Continuous Delivery](https://continuousdelivery.com/) at Auto Trader.

As an application Shippr accepts configurations and artifacts into
its datastore and organises them as 'stacks' that constitute an
application or service.

It can then create 'deployments' of those stacks by instantiating the
infrastructure necessary to realise that stack and provide the app or
service. Shippr will then manage changes to that infrastructure over
the deployment's life until it is decommissioned.

Shippr itself is a Java [Dropwizard](http://www.dropwizard.io)
application using [Amazon S3](https://aws.amazon.com/s3/) for binary
storage, and [MongoDB](https://www.mongodb.com) for storing everything
else.

## Our Issue

Shippr serves an API for communication with other applications /
components, one consumer of which is our health-check dashboard
Dozor. Dozor constantly checks and reports on the status of all of
Auto Trader's applications.

Dozor polls Shippr regularly in order to stay up-to-date with the
latest deployment changes to the cloud platform. Because deploying a
new version of an app involves switching traffic over to a new set of
servers and then terminating the old ones, if Dozor isn't kept updated
false negatives occur as it reports health-check failures for the
terminated nodes.

Unfortunately this is exactly what started happening to us as our API
response times started to grow and grow as our datasets got larger and
larger. Oh dear.


## Investigation

We had already been receiving monitoring alerts telling us that we
were encountering high numbers of '`COLLSCAN`'s for several of our
queries particularly around our `fleets` collection.

Documents in the `fleets` collection actually hold data for multiple
API endpoints, all of which are in Dozor's critical callpath, so it
seemed a good place to start.

### Modelling our Data

A Shippr 'fleet' is a collection of virtual machines all in the same
cloud-environment, a fleet then has 'arrays' that all share the same
role and configuration, and are made-up of 'nodes' which are the
individual VMs themselves.

A full 'deployment' consists of multiple fleets one for each data
centre.

Fleets end up having a subtly different data model across the layers
of our application: API, business logic, and storage.

![Diagram of Shippr's fleet data models]({{ site.github.url }}/images/2017-05-01/fleet-data-models.svg)  

By the time we get to the API what were once just components of fleet
now need to be addressed as fully-fledged discrete resources, this
means that when querying at the datastore layer we end up needing to
locate a fleet in 3 different ways in order to support the look-ups
that the API needs:

 * Find the fleet directly via its ID.
 
 * Find the fleet which contains a particular Array Instance via the
   instance's ID.
 
 * Find the fleet which contains a particular Node via the node's ID.

So here's a (simplified) example of what a `fleet` document looks like
in MongoDB:

```
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

### Checking Query Performance
 
To gauge how effective a query will be MongoDB can be asked to detail
its 'query plan' by calling `.explain()` on the result of a query (the
docs for this are here:
[method/cursor.explain()](https://docs.mongodb.com/manual/reference/method/cursor.explain/))  

so in the `mongo shell` we enter :

```js
db.fleets.find({_id : "some value"}).explain()
```

and we get back:

```json
{
    "queryPlanner" : {
        "plannerVersion" : 1,
        "namespace" : "shippr.fleets",
        "indexFilterSet" : false,
        "parsedQuery" : {
            "_id" : {
                "$eq" : "some value"
            }
        },
        "winningPlan" : {
            "stage" : "IDHACK"
        },
        "rejectedPlans" : []
    },
    "serverInfo" : {
        /* OMITTED */
    },
    "ok" : 1.0
}
```

The key part for us here is the `winningPlan` section - it shows that
the `IDHACK` strategy has been chosen. So what is `IDHACK`? It means
MongoDB will use a specially optimised look-up that can only be used
if the query solely consists of checking the `_id` field. (`_id` is
special and MongoDB will always generate one if you don't provide it
so that it can create this look-up)

From our alerts we're here looking for `COLLSCAN` queries. These are
queries where there was no index to do a look-up in, so MongoDB had to
visit every document in turn to see if it matched.

As a collection gets bigger and bigger these queries become very
expensive as mongo loads the entire dataset in order to query it.

So let's turn to our other queries to see how they fare.

Finding the fleet that contains an array instance:

```js
db.fleets.find({ arrayInstances.someArrayId : { $exists: true } }).explain();
```

```json
"winningPlan" : {
    "stage" : "COLLSCAN",
    "filter" : {
        "arrayInstances.someArrayId" : {
            "$exists" : true
        }
    },
    "direction" : "forward"
}
```

Bingo. Not finding any indexes to speed this up MongoDB has resorted
to checking every fleet to see if it matches. We also have the same
issue with our node query.

```js
db.fleets.find({ nodeIdentifiers : "someNodeId" }).explain();
```

```json
"winningPlan" : {
    "stage" : "COLLSCAN",
    "filter" : {
        "nodeIdentifiers" : {
            "$eq" : "someNodeId"
        }
    },
    "direction" : "forward"
}
```

## In Summary

Here we introduced our application and a performance issue it was
facing and discussed one of the tools provided by MongoDB that enables
us to explore the expected performance of our documents and queries.

In the next post we'll discuss the steps we took to resolve the issue
with our document structure and what we learned about structuring data
in mongo as a result.
