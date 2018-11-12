---
layout: post
title: Delve into our data universe event—November 21st
author: Elly Linnegar
tags: [Tech Talk, Data Platform, Spark]
---

Building on our previous Open Evening of tech talks, we want to delve further into our data universe. What good is having all this data if our people can’t easily access and glean insight from it? Our speakers come from the disciplines of Data Analysis, Data Science and Data Engineering. What brings us all together is our data platform. Find out about the rewards and challenges of delivering on Auto Trader's self-serve data mission.

[Register here](https://www.eventbrite.co.uk/e/delve-into-our-data-universe-tickets-52358871795)—food and drinks will be provided.

<a href="https://www.eventbrite.co.uk/e/delve-into-our-data-universe-tickets-52358871795" target="_blank">
    <img src="{{ site.github.url }}/images/2018-11-12/nov-tech-talks.png" width="70%" class="u-p-10 u-center-img" alt="Tech talk event">
</a>

Below is a summary of the evening’s talks and there will be follow-up blog posts soon that dive deeper into some of the talk topics. We hope to see you there!

#### Selecting ‘attractive’ adverts by modelling click-through rates at scale—Jenny Burrow

In this talk, Jenny will describe how we have utilised our data platform to model click-through rates on the Auto Trader UK website to estimate the ‘attractiveness’ of adverts to our users, and to select which adverts to display. 

#### “Your call is important to us… please hold while we analyse this”—Tom Collingburn

Tom would like to demonstrate how Python & Databricks can enable anyone (even a Python beginner like himself) to identify meaningful topics in unstructured data and visualise the results so that non-data consumers can use the analysis to support business decisions. 

#### Lessons from the Lake—Joe Mulvey

The Auto Trader Data Platform is at the core of our ‘self-serve’ philosophy of empowering business teams to make data-driven decisions. Building it has required us to make choices about architecture, design and technology. How did we do? We take a look back at three critical decisions.

#### Scheduling ETL jobs as code using Airflow—Sam Wedge

We have many ETL jobs pulling data into our Data Lake from various databases, APIs and Kafka topics. We then process that data to ensure a consistent schema, filetype, structure and to remove sensitive information. On top of this, we have refining processes to prepare the data for specific use cases: joining datasets, calculating new fields, removing old fields, deduplicating rows, generating aggregate metrics, performing integrity checks etc. Each of these processes has different requirements to run, which could be dependent on the time of day, the existence of previous data or the successful completion of a previous task. This talk shows how we manage these tasks with a testable and reliable scheduling system with alerting, all configured through code.

*[ETL]: Extract, Transform, Load
