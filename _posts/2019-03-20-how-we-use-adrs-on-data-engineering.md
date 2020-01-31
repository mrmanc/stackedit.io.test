---
title: How we use Architectural Decision Records (ADRs) on Data Engineering
date: 2019-03-20 00:00:00 +00:00
tags:
- Data Platform
- Productivity
layout: post
author: Paul Doran
---

When I joined Data Engineering over a year ago they had already adopted Architectural Decision Records (ADRs) to document architectural decisions made whilst building Auto Trader's Data Platform. ADRs are listed in ThoughtWorks' [Technology Radar](https://www.thoughtworks.com/radar/techniques/lightweight-architecture-decision-records) as a technique to adopt; Data Engineering was the first team I've worked on to use them. They have allowed us to capture the context and consequences of the decisions we make; in a way that provides transparency and allows the whole team, and wider organisation, to contribute.

## What is an ADR?
ADRs are short text files that capture a single decision; see [here](http://thinkrelevance.com/blog/2011/11/15/documenting-architecture-decisions) for a more in-depth description. The ADRs are numbered sequentially and monotonically; with no number reused.

We have adopted the below convention for our ADRs:

* **Title**: A descriptive title in the following format: `<ADR_ID - <Title>`

* **Status**: It must have one of the below statuses.
  ![ADR State Diagram]({{ site.github.url }}/images/2019-03-20/adr_state_diag.svg){: .center-image}
  ADRs can only change status along the permitted transitions shown in the diagram above.

* **Context**: The facts about why we need to make this decision. For example, they could be technological, such as we need to provide a certain capability; or they could be legal, such as compliance requirements.

* **Decision**: The actual decision. We provide a description of what we decided and provide justifications if needed. The decision should be an action we will undertake.

* **Consequences**: We provide a description of all the potential consequences; be they good, bad or neutral.

## How are ADRs used in Data Engineering?

The Data Engineering team maintains a squad website in [GitHub Pages using Jekyll](https://jekyllrb.com/docs/github-pages/). This was already used to share common links, jargon bust and build statuses - so the ADRs were placed there too. The squad website is readable by anyone at Auto Trader and contributions are welcome from anyone.

There are ADRs covering a wide range of subjects; how to partition data in the data lake, choice of business intelligence tool, proposed changes to metric algorithms, etc.

Having a place to record the ADRs is not enough. Consensus needs to be reached on whether ADRs should change status or not. Data Engineering either address this at standup or with specific meetings for more in-depth discussions.

As the Data Platform was built lots of architectural decisions were made. Data Engineering has found it worthwhile to revisit the decisions and see if they still hold; given [what we've learnt]({{ site.baseurl }}{% post_url 2018-11-21-lessons-from-the-data-lake-architectural-decisions %}) as the platform grows.

### Benefits of ADRs

#### One: Clearly reasoned decisions

The format of the ADRs enforces consistency in approach to documenting decisions. This makes it easier for both the author and the reader. The author knows what information to provide to produce an ADR, and the reader knows what to expect when reading an ADR and where to find it.

As the Data Platform is being built new technology is continuously adopted.  The ADRs allow us to show stakeholders that we are in control of technology adoption and change.

#### Two: Good for new team members

I found the ADRs invaluable when I first joined Data Engineering. They allowed me to get up to speed quickly on why the team had made certain decisions. I could read the ADRs to gain a broad understanding of the architecture and to understand the context of the decision. The ADRs told me the history of the decisions made by Data Engineering.

#### Three: Version control for the history of team decisions

Given we are using GitHub Pages to store our ADRs we get a versioned history of their changes for free. I have found it useful to review the history of an ADR to understand its evolution or to know who to ask for clarification if something is unclear.

#### Four: Keeping them lightweight

Data Engineering doesn't embrace ‘big design up front’ so the ADRs are not mammoth documents that nobody will ever read. The theme used for Jekyll has a built-in reading time estimator; the longest ADR to date has an estimated reading time of 4 minutes.

### Challenges of ADRs

#### One: Deciding when to create one

There are no defined rules for when to create an ADR. Individuals are free to propose an ADR when they feel one is necessary. The hope being that by doing this we are capturing all the ADRs needed. I think over time, and with experience, Data Engineering has gotten better at ensuring ADRs are written when required. There have been only a few occasions where I have felt that an ADR was missed.

#### Two: Keeping them up-to-date

The environment Data Engineering work in is reasonably fast paced and lots of decisions can get made in a short space of time. I have found, on occasion, that the ADRs are out of date. 

If we fail to write them in a timely manner then we reduce the value of the ADRs; future potential readers have no way of knowing what was not recorded. This was one of the reasons Data Engineering now have semi-regular catch-ups to discuss ADRs.

#### Three: Keeping them lightweight

The preference for keeping the ADRs lightweight can make them challenging to write. When writing an ADR you want to provide enough information to a future reader, but do it in a concise way. This requires good clear writing; which isn't the easiest thing to achieve when documenting a detailed technical decision.

## Conclusions

Data Engineering has found great value in ADRs as both a historical record of decisions and a way to capture the thinking behind architectural choices. Their number continues to grow.

They were incredibly useful to me when I first joined and I continue to find them a valuable tool.
