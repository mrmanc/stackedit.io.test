---
layout: post
title: Running Experiments At Auto Trader
author: John Payne & Rob Chisholm
tags: [Experiments]
redirect_from:
- /2019/04/18/running-experiments-at-auto-trader.html
comments: true
---

For a number of years, we A/B tested new [www.autotrader.co.uk](https://www.autotrader.co.uk/) features on a separate production environment.  This allowed us to validate that new features had the desired effect, and didn’t negatively impact any of our key indicators with a reduced slice of our users.  But, it required three deployments of our largest web application and was slow and clunky to set up.  We have since introduced a new A/B test platform which allows us to be more agile and respond to change more quickly.  We use the data collated from these tests to drive our next generation features for the site. Read on to find out how we used to test features and why we made this change.

### Previous A/B Testing Solution

Auto Trader's test platform for our website had for several years utilised three production environments (called alpha, beta and gamma) to which application code could be deployed. Alpha and beta would get the latest version and gamma would get the version with new features to test switched on. A small percentage of traffic would then be assigned equally to beta (a control group) and gamma (our test group), enabling user behaviour to be logged and compared. This strategy had many drawbacks:

* **Limited user groups**: beta and gamma had a limited set of servers compared to alpha, so only a small percentage of overall traffic could be assigned to test
* **Test backlog**: there was a large backlog of tests—only one test variant at a time could be run, and full application deployments were needed for each test change
* **Single variant**: testing was limited to A/B testing (i.e. only one test variant)
* **Mobile readiness**: the environment was only built for the desktop version of the website, so was not available for the (rapidly growing) mobile versions

Auto Trader needed a better solution:

* To enable multivariate testing (i.e. more than one test variant)
* To accommodate more than our core website
* To permit multiple concurrent tests in each system
* To permit tests to be enabled and disabled without a deployment
* To permit the flexible allocation of users to tests

### Replacement A/B Testing Solution

We found that Indeed Engineering’s open-source [Proctor](http://opensource.indeedeng.io/proctor) framework offered a robust and easily adaptable solution. A call to Proctor with a unique id (e.g. a user id) will guarantee allocation to test groups, based on percentages or rules specified in configuration files.

Proctor can utilise two easily configurable JSON files to manage allocations of users to test groups: ***specifications*** and ***definitions***.

Test specifications define the test names, test buckets and (rule) contexts. If, for example we need a test group with two test variants, we define four buckets (one for non-test traffic, two for the variants and one for control).

```json
"buttonColourTest": {
  "buckets": {
    "group0": 0,
    "group1": 1,
    "group2": 2,
    "group3": 3,
    "fallback": 999
  },
  "fallbackValue": 999
}
```

(‘fallback’ is a Proctor exception-handling group which has no user allocations.)

Test definitions define test descriptions, rules and allocations. For our test example above, we can allocate 70% of users to the non-test (inactive) group and 10% to each of the test and control groups:

```json
"allocations": [{
  "rule": null,
  "ranges": [{
    "length": 0.70,
    "bucketValue": 0
  },
    {
      "length": 0.10,
      "bucketValue": 1
    },
    {
      "length": 0.10,
      "bucketValue": 2
    },
    {
      "length": 0.10,
      "bucketValue": 3
    },
    {
      "length": 0,
      "bucketValue": 999
    }]
}]
```

It is also possible to segment users based on rules specified in a test definition. For example, logged-in or logged-out status can drive users to test groups targeting those user types. The Proctor documentation provides examples of rule-based allocation.

Auto Trader has encapsulated Proctor’s capabilities in a SpringBoot application named Test Allocator. The Proctor demo class DefinitionManager provided a useful starting point from which to investigate and extend functionality to fit our requirements.  We then created separate endpoints for each system requiring user allocations, so that each system has an independently maintained set of test specifications and definitions.

Once deployed, a system calls an endpoint on our Test Allocator service, with unique user id as a parameter. Test Allocator responds with a list of groups to which that identifier has been assigned. For example:

```json
"groups": [
  "buttonVariant2",
  "test2control",
  "test3inactive"
]
```

We can then write code to switch functionality based on the user’s test group. For example:

```java
switch(usersAssignedGroup) {
  case "buttonControl":
    buttonColour=BLUE; …
  case "buttonVariant1":
    buttonColour=RED; …
  case "buttonVariant2"
    buttonColour=GREEN; …
…
```

That user’s behaviour can then be tracked to enable comparison of control and test variants.

Using the Proctor framework has greatly enhanced Auto Trader’s test capabilities, enabling rapid and concurrent multivariate testing across a range of systems.

### Learnings

Building a new experiments platform has not been without a learning curve. The technical improvements we made allowed us to do much more and we had to look at our practices around how we run experiments.

The temptation to run multiple tests at once can cause issues when running concurrent tests in the same area, as we found out when running a new full page advert test alongside a rework of our live chat solution. Doing too much at the same time can invalidate your tests, as user behaviour is affected by testing two new features on the same page at the same time.

Ideally, you should test one feature in a particular area at a time as a rule of thumb. Within that test group, you can test multiple designs and copy to validate your test hypotheses.

In the instance above the live chat was covering some of our lead metrics which invalidated our tests for the full page advert changes we were making, as we were seeing a drop in events attributed to this call to action.

This can lead to confusion and wastes time as we have to get a data analyst to look into the recorded metrics, but wrong assumptions can be made.  A drop in metrics can be attributed to the wrong test when another one could be causing this drop.
This could in theory cause you to fail a test for the wrong reasons.

### Tidying up

When the tests are set up and allocations assigned the enabled test group values are added to a test bucket cookie.  We then pass this cookie value string to our various tracking tools like Google Analytics and our internal Operational Data Store (ODS) system, which enables us to filter test behaviour and derive outcomes.

Towards the end of 2018 it was found we were having a problem with recording data for our latest tests. We found that there was no data in a couple of instances.

After some investigation, we found that we were recording the cookie value in a field called `experiment_ids` which had a max character value of 125 characters.  Our test string length being passed at that time was 167 characters so we were losing test data for five active test groups—not an ideal scenario.

So we started a period of tidying up, closing down old inactive tests. We also introduced a character limit on test values which enabled us to avoid breaching the 125 character limit imposed in our internal ODS system.

We made changes to improve the visibility of what tests were active and highlighted the age of those tests. This helps us when we are investigating issues. We made sure that tests changed colour as they aged (90+ days shows up red) to highlight where we need to have conversations with teams about decommissioning tests groups.

<figure>
<img src="{{ site.github.url }}/images/{{ page.date | date: "%F" }}/old-ab-allocator.png" width="70%" class="u-p-10 u-center-img" alt="Screenshot of the initial prototype Test Allocator Dashboard">
<figcaption class="u-center-img">Initial prototype Test Allocator dashboard</figcaption>
</figure>

We already recorded which team ‘owns’ test groups but as we'd been through a team restructure in early 2018 this needed looking at to establish which new teams were responsible for decommissioning. We introduced an ownership filter to help teams see what they were responsible for.

We created an alert to notify us if the current size of the `experiment_ids` field in the cookie (which stores the enabled tests) would trigger our ODS character limit issue, ensuring that we don't lose any test data in the future.  We plan to make changes to our Continuous Integration process to provide us with faster feedback if a change breaches this limit.

<figure>
<img src="{{ site.github.url }}/images/{{ page.date | date: "%F" }}/new-ab-allocator.png" width="70%" class="u-p-10 u-center-img" alt="Screenshot of the improved Test Allocator dashboard">
<figcaption class="u-center-img">Improved Test Allocator dashboard</figcaption>
</figure>

### Future Tests

With the groundwork down around improving the way we use our A/B test environment, we feel it’s now easier to use and see which tests are currently in flight.  This should help to prevent tests clashing in the future.

One additional improvement we may introduce is some information about where on the website the test is being carried out, e.g. ‘homepage’, ‘nav’, ‘full page advert’. This should ensure we have a view of tests that are active in a given area across all the teams, further improving visibility.

### Summary

Although there were some stumbles along the way, building ourselves a new experiments platform based on existing technologies has left us in a much better position. We are now able to validate new functionality more quickly and with less investment, helping us to improve our products in a way that is necessary for the marketplace we exist in. The complexity of our technical architecture meant that the shared approach we followed was worth the investment. 