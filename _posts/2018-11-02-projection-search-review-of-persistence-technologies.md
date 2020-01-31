---
title: 'Projection Search: Review of Persistence Technologies'
date: 2018-11-02 00:00:00 +00:00
tags:
- REST
- Search
layout: post
author: Neil McLaughlin
---

Having [previously]({{ site.github.url }}{% post_url 2018-09-19-projection-search %}) established what we mean by a projection search, it's time to have a think about how we might build it. Let's start by reviewing the current landscape of Java persistence technologies, comparing and contrasting these, with an eye on suitability for our search. We're not going to cover all of these, but we are going to cover a representative sample. 

We will refer here to [Hibernate](http://hibernate.org/), but many of the points made will apply equally to other Object-relational mapping frameworks.

## JDBC
JDBC is the fundamental Java framework for database access. With JDBC you execute SQL statements which are represented as strings within the code.

For some niche applications managing small queries this is okay. More generally, managing SQL statements as strings within a Java application is a grim and thankless task. Without any type-safety, it is hard to spot simple errors in SQL statements when they’re all wrapped up in multiline Java Strings.

Consider the example below in the context of an application in which emerging new requirements may require splicing together strings to conjure up different variations of the query with various where-clauses, optional sorts and possible joins and you will probably start to form a picture of why JDBC fell into disuse.

```java
        String advertiserQueryById = "select a.id," +
                "  a.ADDRESS1," +
                "  a.ADDRESS2," +
                "  a.ADDRESS3," +
                "  a.TOWN," +
                "  a.COUNTY," +
                "  a.POSTCODE," +
                "  a.LATITUDE," +
                "  a.LONGITUDE," +
                "  a.NAME," +
                "  a.FIRST_CREATED," +
                "  a.LAST_UPDATED," +
                "  a.EMAIL_ADDRESS," +
                "  a.ADVERTISER_TYPE," +
                "  a.INTERNATIONAL_CODE," +
                "  a.AREA_CODE," +
                "  a.LOCAL_NUMBER" +
                "from advertiser a" +
                "where a.dealer_id = ?";
        PreparedStatement preparedStatement = connection.prepareStatement(advertiserQueryById);
        preparedStatement.setString(1, "sample-dealer-id");
        ResultSet resultSet = preparedStatement.executeQuery();
```

## Object-Relational Mapping—Hibernate
[Hibernate](http://hibernate.org/) is a framework for mapping an object-oriented domain model to a relational database. Mappings are defined declaratively (annotations or XML if you prefer) and the Hibernate framework provides data retrieval and query facilities.

```java
public StockItem getStockItemById(final String id) {
        return this.entityManager.find(StockItem.class, id);
}
```
We can also use HQL, an object-oriented query language
```java
List<StockItem> stockItems = entityManager.createQuery("from StockItem s " +
      "where s.advertiser = :advertiser and s.externalClientReference = :externalClientReference " +
      "order by s.lastUpdated desc", StockItem.class)
                .setParameter("externalClientReference", clientReference)
                .setParameter("advertiser", advertiser)
                .getResultList();
```

What's great about Hibernate is how quick and easy these mappings are to set up—try and write the above code manually and you will see what I mean. While it's very easy to get an application up and running in this manner, there are a lot of notorious complexities around performance tuning of Hibernate applications.

Hibernate is a deeply seductive technology. Object-relational mapping annotations make it very easy to get a persistence layer up and running. It's not quite that simple though. This kind of default setup is not necessarily performant and may require significant tuning to become production-ready. Mapping graphs of objects is a particular source of complexity which can drag you into a murky world of proxy-objects with eager and lazy fetching strategies.

The Hibernate Criteria API is intended for retrieving entities by composing Criterion objects. This is intended for functionality such as searches where there is a variable number of conditions to be placed upon the result set. This sounds like it may be a suitable alternative to the naive approach of rolling our own HQL (Hibernate Query Language) strings dynamically.

The real problem with Hibernate in the context of our search is that Hibernate is designed to get back entire graphs of objects. For example, using the Criteria API or a HQL string we could feasibly search for a StockItem. Hibernate will retrieve that StockItem from the database with all of its attendant adverts and media items. This is great if we want all of that data, but we may only want something simple, such as make, model and whether or not the stock item is advertised.

It's not very good at fetching a limited subset of this data and it can't really do this dynamically at runtime.

## jOOQ—bringing SQL into Java

The authors of the [jOOQ](https://www.jooq.org/) framework believed that the ORM approach hiding the SQL behind a complex abstraction layer was a fundamental mistake. jOOQ compiles the database schema into the project, incorporating both SQL and the schema into the project as an embedded language.


Let's look at a simple Mercury query in both the SQL console and then in jOOQ. The query itself is a very basic Mercury query, selecting make, model and derivative from the STOCK_ITEM table for a given trade advertiser. Trade advertisers are typically referenced by the dealerId.  

```sql
select si.make, si.MODEL, si.DERIVATIVE
from STOCK_ITEM si inner join advertiser a on si.ADVERTISER_ID = a.id
where a.dealer_id = 'sample-dealer-id'
order by si.CREATED_DATE desc;
```

This is the same query in jOOQ. It's quite clearly SQL but also native Java, with all of the benefits that brings, including type safety and auto-completion (of SQL keywords and schema objects) within the IDE.

```java
connection.select(STOCK_ITEM.MAKE, STOCK_ITEM.MODEL, STOCK_ITEM.DERIVATIVE)
                .from(STOCK_ITEM.innerJoin(ADVERTISER).on(STOCK_ITEM.ADVERTISER_ID.eq(ADVERTISER.ID)))
                .where(ADVERTISER.DEALER_ID.eq("sample-dealer-id"))
                .orderBy(STOCK_ITEM.CREATED_DATE.desc())
                .fetch();
```

This approach works well, but there are drawbacks.

Firstly to incorporate jOOQ into the project we need to add an extra stage to the build cycle to compile the schema objects. This is a one-off task, but it is a little tricky and it does ramp up the learning curve.

Having committed to jOOQ and started development, writing DAOs in jOOQ is a little more involved. Developers will need to code the mappings between domain objects and the jooQ ActiveRecord objects generated from your schema. This can be coded manually or using a mapping framework such as [Orika](https://orika-mapper.github.io/orika-docs/). Either way, this will seem tedious to a seasoned Hibernate developer. It is not necessarily a bad thing. We are much less likely to encounter certain kinds of complexities common in Hibernate applications.

Comparisons with Hibernate aside, SQL is a much better choice for a projection search, and jOOQ, which allows us to build SQL dynamically in a type-safe fashion is an obviously good choice here.

## Pros and Cons

|Hibernate...|jOOQ...|
|-----------|------|
|Powerful, popular and an industry standard|Powerful, growing in popularity|
|Requires no build modifications| Requires build modification to compile schema at build time|
|Is Open Source and free|Is Open Source but requires a licence for Oracle and other commercial databases|
|Uses declarative mappings to automate coding of CRUD operations |Requires manual coding of CRUD operations via the ActiveRecord pattern|
|Complex: proxy objects; lazy and eager loading|SQL is a powerful and elegant language|
|Hard to tune performance|Easy to tune as it's (mostly) just SQL|
|Based on a leaky abstraction which can add massive complexity to your application|It's just SQL|


## The Hybridised Approach

Hibernate and jOOQ are powerful frameworks with very different world views. They are also more complementary than you might imagine. As we have already noted, Hibernate is good at the basic CRUD operations at an object level while the use of SQL through jOOQ enables to meet more complex search requirements than Hibernate is designed to support.

The complementary nature of jOOQ and Hibernate is a good argument for a hybrid design principle. An even more compelling argument is that we are planning to add projection search to an existing application with a mature and heavily utilised Hibernate entity model, which we have no immediate wish to rewrite.


## Adding jOOQ to the Application Build
Before we can do anything interesting with jOOQ, we need to integrate it with the application build. For a comprehensive guide, I'd suggest a look at the jOOQ documentation. This is an outline of how we proceeded with our own application stack.

How to go about this will depend somewhat on the database technologies you are using. We use [Flyway](https://flywaydb.org/) to manage our database migrations. Additionally, we use a [H2](http://www.h2database.com/html/main.html) database for local deployments & integration tests. In our case, the steps we took were as follows.

1. Add a flyway task to the build to create a local H2 schema—Flyway ensures this is an up-to-date copy of the schema

1. Add a jOOQ code generation step to compile the above H2 schema  
This will generate a set of Java classes with each class mapping to one of our database tables (or other objects if we like)

1. Add the generated classes to the application classpath

1. Run the build and observe with satisfaction your package of table objects generated by jOOQ

**Note**: The use of Flyway here provides some assurance that the version of the database we are using in dev is the same as that in QA or live environments; otherwise, there would be potential for conflict between the jOOQ code and the actual schema in the QA or live environments.

## Summary
We've looked at a fairly representative range of contemporary Java persistence technologies, concluding that jOOQ is the most suitable for our projection search. In the next blog in the series, we will look at how to go about using jOOQ to implement a query builder which will dynamically create the various queries required by the projection search.
