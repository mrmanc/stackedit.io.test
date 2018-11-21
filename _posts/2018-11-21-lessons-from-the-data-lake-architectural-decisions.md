---
layout: post
title: 'Lessons from the data lake, part 1: Architectural decisions'
author: Paul Doran, Sien Figoureux, Joe Mulvey, David Whittingham
tags: [Data Platform, AWS, Spark, Avro]
---
It's eighteen months since the inception of the Auto Trader Data Platform, a cloud-based storage lake and computing solution designed to unlock the power of Auto Trader's huge and growing data set of vehicle transactions in the UK. It's time to ask, what have we learnt, what have we changed, and did we deliver what we intended?

<img src="{{ site.github.url }}/images/2018-11-21/data-lake-monster.png" width="70%" class="u-p-10 u-center-img" alt="Data lake monster">

Auto Trader has been in the business of serving out web-based vehicle advertising since 1997. Making effective use of our data has been a significant priority since we became an exclusively digital business in 2013.
In 2017, it became clear that we needed to upgrade our tried and trusted data analytics platform, based on a warehouse model, into something more Agile and responsive to business needs. In late 2018, that new platform is a working reality, a data lake built with [S3 storage](https://aws.amazon.com/s3/) and distributed computation with [Apache Spark](https://spark.apache.org/). Building it has been challenging, involving the adoption of new technologies, new computing paradigms and embracing the opportunities and risks of open source software.

In this post, we will review some of the high-level architectural choices we made and their sometimes unexpected consequences. Future posts will cover specific software solutions we have adopted, the behavioural changes the business has had to make to adopt the platform, and some of the technical gotchas that have tripped us up over the past year and a half.

## Cluster computing on-premises is hard and expensive, cloud is easier

Late in 2016, we began work on our first incarnation of a data lake. We started by building an on-premises virtualised data platform using [Apache Ambari](https://ambari.apache.org/), which is a Hadoop management system for provisioning, managing and monitoring Hadoop clusters. We chose the Hadoop Distributed File System (HDFS) for storage, Apache Spark as the compute layer, resource scheduling using [YARN](https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html) and ingestion via [Apache Flume](https://flume.apache.org/).

A series of operational problems confronted us. First, and most pressing, was the question of how much storage and computation capacity to build. One of the weaknesses of the "on-prem" approach is that it naturally encourages over-provisioning. Businesses need to provide for the peak usage case, rather than the average, and no-one wants to be responsible for failing to provide enough. The result is excess capacity.

Even if we could answer the question of "how much?", the question of "how?" remained. Should we separate storage and compute, or have them on the same machine? How many disks would we need, and of what capacity? Do we emphasise CPU or memory when provisioning boxes? The answer to these questions may be specific to the usage patterns of the business, but experimentation with hardware is an expensive hobby.

In the end, after arriving at answers to some of these questions, we found the operational maintenance of the Hadoop cluster to be extremely intensive. Just keeping track of disk failure and effecting repairs would be a full-time job.

<img src="{{ site.github.url }}/images/2018-11-21/cloud-image.png" width="50%" class="u-p-10 u-center-img" alt="cloud image">

In short, our attempt at creating an on-premises Hadoop cluster did not meet our needs, but we learnt many valuable lessons to move forward. What replaced it is a solution built in AWS based on the S3 object store, which provides essentially infinite scalable storage, and Apache Spark delivered through [Amazon's EMR solution](https://aws.amazon.com/emr/), which is elastic, configurable and flexible. Our infrastructure is configured as code using [Terraform](https://www.terraform.io/), giving us the added benefit of version control.

The economics of cloud versus on-premises have been [hotly debated](https://www.forbes.com/sites/forbestechcouncil/2017/07/25/the-cloud-vs-in-house-infrastructure-deciding-which-is-best-for-your-organization/#67ca7dab20f6) but we are clear that for our business the argument is settled. The overall benefits of freeing up developer and operational resources, managing the maintenance and hardware development and (above all) elastic provisioning vastly out-weigh any additional costs.

## A conceptually simple data zoning system has enabled self-service

So, given we have settled on S3 for storage, how do we impose some structure on our lake?

Trawl through the internet for articles on "data lake architecture" and you will find a vast menagerie of diagrams, each prescribing the best way to bring some order. The range of solutions on offer is itself convincing evidence that no canonical solution exists.

However, there are certain themes and most involve progressive refinement of raw data into a state that is useful to data consumers within the organisation. Our structure follows this same principle. It's striking that the diagram below has stayed the same throughout the development of our platform, and its components have become part of the data lexicon at Auto Trader.

<img src="{{ site.github.url }}/images/2018-11-21/lake-structure.png" width="70%" class="u-p-10 u-center-img" alt="Data Lake Zone Structure">

Briefly, we confine all our data to five 'zones' - in practice, five S3 buckets - named transient, raw, refined, user and trusted.

All data enters the lake via the transient zone. It arrives by an automated ingestion process---principally via [Kafka](https://kafka.apache.org/) sinks or [Apache Sqoop](http://sqoop.apache.org/) for ingestion from relational databases---in a variety of data formats and at irregular intervals. An automated ETL process (written in Spark) then moves the data to the raw zone. Crucially, there is no human interaction with the transient zone.

We store data in the raw zone under catalogued prefixes. We have a strict naming policy that enforces the unambiguous partitioning of data according to ingest time, and which allows us to make use of Hadoop semantic partitioning. Other than the organisation of the data and standardisation into a single format ([Avro](https://avro.apache.org/), of which more below), no processing is carried out and the data is unchanged from its state on ingest.

The refined and user zones are where the curating of data into usable packages begins. The user zone is a play area where analysts and developers can save data sets they have built from the source material in the raw zone. Typically, they will use a notebook to execute queries, perform calculations or train a model based on data sets of interest, saving the results in their own named area. This allows them to conduct an ad hoc, one-off study---perhaps to create a simple dashboard---or perform a spike for a production process.

The output of production code goes into the refined zone. Here we store data used for business processes---typically snapshot history of our core business information, or derived data sets or models used by live services. Refined data is usually the result of careful filtering, de-duplication and cleaning, so analysts like reading from this area if a suitable data set exists.

Finally, the trusted zone exists for the most carefully validated and prepared information. The data in this zone should be suitable for regulatory reporting, KPIs or accounting information. There is no restriction on data format, for example, some data may be saved as CSV to allow easy import into spreadsheets.

This simple structure has served us well. It is straightforward to communicate and understand, and has become a major part of the data landscape at Auto Trader. The zoning structure has proved robust, and it is pleasing that our consumers within the organisation are navigating it successfully. It will remain the central pillar of our platform for some time to come.

## Achieving schema-on-read requires good behaviour as well as good tech

A key requirement of the data lake model is to achieve "schema-on-read". The lake holds files in potentially any format. In order to make the data usable, it is crucial to decide on the file format used to store the data, specifically in its unrefined form in the raw zone. We initially chose JSON, but we found several disadvantages:
* Inefficient storage as JSON favours readability over storage size
* No out-of-the-box data partitioning to match block sizes
* No explicit schema

To see how other data formats commonly used in the big data community compared, we chose to evaluate [Avro](https://avro.apache.org/) and [Parquet](https://parquet.apache.org/). Both are well-documented open-source projects that have support in several programming languages and handle partitioning of data well. Where Avro is a row-based file format that's faster at writing data, Parquet is a columnar format, optimised for fast reads and queries. Both formats carry meta-data within their file structure to infer the schema at read time.

Data arrives in the transient zone in a variety of formats, so part of the process of ingestion into the raw zone is to standardise it. We chose Avro for this purpose, as the transient-to-raw migration operates on the full table structure in a manner well suited to the row-based architecture. Conversely, for the refined zone, we chose the Parquet data format as this data is primarily used for handling `SELECT` queries by analysts.

There have been some unexpected consequences arising from our use of Spark as our computation engine. Much of the data we ingest from Kafka is highly structured, nested data-types with nullable subfields. It turns out that Spark's ability to infer schemas for this data is limited.

First, if a schema evolves (by adding a column, for example), then Spark may (under certain circumstances) simply drop the data when combining it with datasets written with the previous schema. Technical workarounds are possible, but transformations need to be carefully written to avoid data loss.

Second, if Spark has to infer a schema from data (e.g. in JSON format) which includes a null field, it will by default infer this as being a `String`. If subsequent data contains data that cannot be cast as a String (perhaps an `Array` type) then the read operation will throw an error.

Of course, specifying a schema overcomes this difficulty, but does not meet the schema-on-read requirement. Furthermore, it leaves us open to the evolution issue: it may be possible to read data with an out-of-date schema as long as all the specified fields are present, but new fields could be silently discarded, leading to data loss.

The conclusion is that schema-on-read in its purest form is hard to achieve, and that it remains necessary to keep tight control of schemas from ingestion onwards. This may require some technological solutions---such as a [schema registry](https://docs.confluent.io/current/schema-registry/docs/index.html)---but it will definitely require behavioural discipline around management of the lake and good communication with the upstream data providers.

## A Data Catalogue is essential, but a low-tech solution goes a long way

A Data Catalogue centralises all the information on the data in the lake in one location; which enables end users to self-serve. Initially, we had a small number of data sets and the lack of a catalogue was not impacting us. As the data lake became more widely adopted the number of data sets we ingested increased and the need for a Data Catalogue became apparent.

We considered the following three options:

### Option 1: buy a commercial solution

We evaluated several commercial solutions; most of which met our requirements. However, we felt that they were either too expensive or would have required significant changes to our existing data lake processes. We anticipate the number of data sets in the data lake to continue to grow and as such, we may feel in the future that the commercial solutions offer better value for money.

### Option 2: build our own

Given we decided a commercial solution was not viable we considered building our own. There's a definite advantage to building a solution yourself, as you can get exactly what you want. However, the amount of effort and resource you wish to invest in building your own Data Catalogue needs to be taken into consideration and we felt that the trade-off was not worth it. We instead focused our efforts into extending the data platform capabilities; which has ultimately delivered more value to the business.

### Option 3: adopt a low-tech solution

Despite ruling out the above two options we still needed a Data Catalogue so we settled on a lower-tech solution. We have created a wiki page that we manually curate when new data sets are ingested or when data sets are changed. The wiki page is proving useful for surfacing our data sets to the wider business and for teasing out the requirements we have of a Data Catalogue. It comes with the obvious overhead of needing to be manually edited, meaning that it is occasionally stale. We recognise there will be a tipping point where this solution becomes unsustainable and we'll need something more sophisticated, but by that time we will know exactly what we want our Data Catalogue to do.

The need for and value of a Data Catalogue were clear and there were several plausible solutions. We've found that our low tech solution has proven useful and capable.

## Conclusion
To conclude:
* The "cloud" vs "on-prem" decision has been vindicated so thoroughly that it seems odd to think there was ever a debate about it. Situating our platform in the cloud allows us to actively and nimbly re-provision and scale our hardware in a way that would be unthinkable in our own data centres. Operational deployments are faster by several orders of magnitude.
* Clear and unambiguous zoning has also proven valuable but has brought its own technical challenges and gotchas. Maintaining a workable structure takes behavioural change and active engagement, especially in a post-GDPR world.
* Schema-on-read in its purest form is not easily achievable. As well as technological solutions, internal data providers need to be aware of the lake as a significant downstream consumer and be prepared to assist with schema maintenance.
* Data cataloguing is crucial, but a fully featured solution is either difficult or expensive. A low-tech option may be sufficient in the short term.

In our next post, we will talk about some of the specific solution choices - good and bad - that we made in attempting to build our self-service data platform, and what gaps we still feel remain to be filled.

