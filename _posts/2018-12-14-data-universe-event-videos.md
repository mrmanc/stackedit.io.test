---
layout: post
title: Data Universe event videos
author: Elly Linnegar
tags: [Tech Talk, Data Platform, Spark, Airflow, Data Science]
---
It's been a few weeks now since our last public tech talks event---a big thank you to all who came. Thanks also for all the feedback we received on the evening. We plan to use this to help shape our next event. For anyone who missed the event, please find the videos of all the talks below. Watch this space for details of our next event in the new year.

## The videos

#### Lessons from the Lake—Joe Mulvey

The Auto Trader Data Platform is at the core of our ‘self-serve’ philosophy of empowering business teams to make data-driven decisions. Building it has required us to make choices about architecture, design and technology. How did we do? We take a look back at three critical decisions.

<iframe src="https://player.vimeo.com/video/305918775" width="640" height="360" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen class="u-p-10 u-center-img"></iframe>

<br/>

#### Selecting ‘attractive’ adverts by modelling click-through rates at scale—Jenny Burrow

In this talk, Jenny describes how we have utilised our data platform to model click-through rates on the Auto Trader UK website to estimate the ‘attractiveness’ of adverts to our users, and to select which adverts to display. 

<iframe src="https://player.vimeo.com/video/305922647" width="640" height="360" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen class="u-p-10 u-center-img"></iframe>

<br/>

#### Scheduling ETL jobs as code using Airflow—Sam Wedge

We have many ETL jobs pulling data into our Data Lake from various databases, APIs and Kafka topics. We then process that data to ensure a consistent schema, filetype, structure and to remove sensitive information. On top of this, we have refining processes to prepare the data for specific use cases: joining datasets, calculating new fields, removing old fields, deduplicating rows, generating aggregate metrics, performing integrity checks etc. Each of these processes has different requirements to run, which could be dependent on the time of day, the existence of previous data or the successful completion of a previous task. This talk shows how we manage these tasks with a testable and reliable scheduling system with alerting, all configured through code.

<iframe src="https://player.vimeo.com/video/305925502" width="640" height="360" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen class="u-p-10 u-center-img"></iframe>

<br/>

#### “Your call is important to us… please hold while we analyse this”—Tom Collingburn

Tom demonstrates how Python & Databricks can enable anyone (even a Python beginner like himself) to identify meaningful topics in unstructured data and visualise the results so that non-data consumers can use the analysis to support business decisions. 

<iframe src="https://player.vimeo.com/video/305928040" width="640" height="360" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen class="u-p-10 u-center-img"></iframe>

<br/>

*[ETL]: Extract, Transform, Load
