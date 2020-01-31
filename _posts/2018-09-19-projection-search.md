---
title: 'Projection Search: Introduction'
date: 2018-09-19 00:00:00 +01:00
tags:
- REST
- Search
layout: post
author: Neil McLaughlin
excerpt_separator: "<!--more-->"
---

What do we mean by Projection Search? Almost all web applications need to provide their user with a search, providing a summarised view of their data. An email inbox is a good example of this. Within AutoTrader, we need the ability to list a user's vehicles and adverts.   

Usually, this type of search returns a matching list of resource summaries, each summary being a subset of the actual resource, with some mechanism - such as a link - for finding the resource. 

Our recurring design headache is to decide _which fields are included in the summary object._

<!--more-->

```json
[
  {
    "id": "021",
    "make": "Ford",
    "model": "Fiesta",
    "_links": {
        "self": {
            "href": "https://example.com/stock/021"
        }
    }
  }
]
```

We could provide the entire stock item, dropping the notion of a summary entirely. The disadvantage is one of load and performance - our application would have to extract and serialise a wealth of information from the database, most of which simply isn't needed. In addition, the sheer volume of data transmitted would in itself negatively impact performance.

We could go minimal and provide simply a link to each resource. This would enable us to write a very simple (and fast) search. It would not be a very useful service. The user would have to crawl to each resource object in turn, slowing down their service and (once again) generating significant load on our service as we retrieve every resource.

Usually, we take an educated guess, providing what we genuinely believe to be a suitable summary for most users. Of course .... requirements change... new products evolve and our summary inevitably becomes obsolete.

In the end, there will always be some user who has good reasons for wanting one extra field in the summary. The idea of a projection search is that _each user gets to specify what they would like in their result set_.

This series of blogs will talk you through our recent implementation of a projection search.


## Our Stock Service ... and our search requirements
Central to the Auto Trader platform our stock service processes thirty thousand new stock items daily and many more incremental updates besides. Incoming stock items and their adverts are cleaned up, standardised against the Auto Trader taxonomy, screened for unsuitable content, persisted to a database and finally published to our search platform.

Typical usages of our service include
* Creation or update of Trade or Private advertiser.
* An advertiser searches for their stock and adverts.
* An advertiser creates, reads, updates and deletes stock and adverts.

Our service is used by other squads within Auto Trader who build and maintain the applications vehicle retailers use to maintain and advertise their stock.

These vehicle retailers include customers of both Auto Trader and our Irish business Carzone. They range from private advertisers selling the family car, small and medium car dealerships, larger franchises, manufacturers, truck, plant, farm retailers as well as caravan and motorhome specialists.

### Domain Model 
Our domain model consists of **Advertiser**, **StockItem** and **Advert** objects. An **Advertiser** may have zero or more **StockItem**, which may have various types of **Advert**.


### The Stock Resource
This is an excerpt of a **Stock** resource (the real version has lots more fields) – the stock item exposed by the restful service in response to `GET /api/stock/2c9299d15eaa22a1015eaa5fac367828`

```json
{
    "identifiers": {
        "_id": "2c9299d15eaa22a1015eaa5fac367828"
    },
    "registration": {
        "registration": "GU64KNS"
    },
    "webAdverts": {
        "consumerWebsite": {
            "startDate": "2017-09-22T00:00:00+01:00",
            "endDate": "2017-10-25T16:50:39+01:00",
            "status": "COMPLETE",
            "_advertised": true
        },
        "dealerWebsite": {
            "startDate": "2017-09-22T00:00:00+01:00",
            "endDate": "2017-10-25T16:50:39+01:00",
            "status": "COMPLETE",
            "_advertised": true
        },
    },
    "derivativeInformation": {
        "make": "MAZDA",
        "model": "3",
        "bodyType": "Hatchback",
        "doors": 5,
        "transmission": "Manual",
        "fuel": "Petrol",
        "vehicleType": "Car",
        "derivative": "2.0 165 Sport Nav 5dr"
    },
    "_advertiser": {
        "dealerId": "12345"
    }
}
```

From the data, you will be able to see that the stock item in question is a car and that the data is organised within various categories: identifiers, registration, webAdverts, derivativeInformation, advertiser. Fields prefixed with an underscore are read-only.

### Projection Search Requirements

Our clients require that we provide a performant search mechanism for StockItems which provides
- Consistent ordering of results 
- Filtering results by dealer id (with other filters being added as necessary)
- Configurable result set allowing any of the following fields 
  - id (**identifiers.\_id**)
  - registration (**registration.registration**)
  - make (**derivativeInformation.make**)
  - model (**derivativeInformation.model**)
  - derivative (**derivativeInformation.derivative**)
  - flag indicating item is advertised on consumer website (**webAdverts.consumerWebsite.\_advertised**)
  - flag indicating item is advertised on dealer website (**webAdverts.dealerWebsite.\_advertised**) 

We can add our own further stipulations. 
- The search should be exposed as a GET request, filter parameters and projection fields being specified within the URL.
- The search should scale as we add future projection fields.
- A response value is provided for each field in the request _and only those fields in the request_.
- Wildcard type operations are not supported - requesting derivativeInformation.\* is disallowed.


### Request and Response
The search endpoint will accept a filter parameter and a set of 'includes' parameters, each specifying a projection path. 

For example, this simple search requests only the ids of stock items belonging to the specified dealer. 

`https://example.com/api/stock?_advertiser.dealerId=12345&includes=identifiers._id`

A more ambitious search requests id, make, model and advertised status (on autotrader.co.uk) 

```
https://example.com/api/stock?
_advertiser.dealerId=12345
&includes=identifiers._id,
&includes=derivativeInformation.make,
&includes=derivativeInformation.model,
&includes=webAdverts.consumerWebsite._advertised
```

### Search Response
Search response will be a JSON document containing a set of key-value pairs. The keys will be the projection paths (eg: derivativeInformation.make). The values will be the values of those fields in the underlying stock resource. 

For the simple request of just the ids
```json
[
 {
  "identifiers._id": "2c968d2d5eecb32a015eecbc303a02ac"
 },
 {
  "identifiers._id": "3272ffd826ee47ae853cb2ce38852b6c"
 }
]
```

For the id, make, model and advertised status (on autotrader.co.uk) search, we get
```json
[
 {
  "identifiers._id": "2c968d2d5eecb32a015eecbc303a02ac",
  "derivativeInformation.derivative": "2.4 D5 SE 4dr",
  "derivativeInformation.model": "S80",
  "webAdverts.consumerWebsite._advertised": false
 }
]
```

## Summary

A projection search enables an end user to explicitly declare which fields they wish to search for. This ongoing set of blog posts tells the story of how we implemented a stock projection search for our Stock service and how it was received by our customers.

The service itself runs on Spring Boot with an Oracle back end. After some [consideration]({{ site.github.url }}{% post_url 2018-11-02-projection-search-review-of-persistence-technologies %}) we settled on a jOOQ based solution which uses a builder to create custom SQL queries as required by the incoming search request. As our story progresses, we will discuss the challenges of handling fields before moving on to increasingly complex joins.
