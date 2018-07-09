---
layout: post
title:  "Speaking about Continuous Deployment at Pink18"
author: Matt Beaney
excerpt_separator: <!--more-->  
---
Last year Dave Whyte and I were lucky enough to be invited to speak at [Pink18](https://www.pinkelephant.com/en-ca/Pink18/Home) in Orlando, Florida.  The Pink Elephant conference is the worlds largest and most prestigious IT Service Management conference, having run for over two decades.  

This years theme was Integrated Service Management, which is about aligning traditional Service Management processes with DevOps and Lean/Agile.

This post is about the talk I gave, the public speaking experience and the preparation required to deliver a cohesive talk.

<!--more-->  

<img src="{{ site.github.url }}/images/2018-07-09/pink18.jpg" style="width: 60em; float: center;" alt="Pink18 Logo">

## Preparation, Preparation, Preparation!
Speaking publicly is not something that should be undertaken lightly.  Especially when speaking at a high profile conference like this!

I spent roughly an hour a day for six months, first developing the presentation itself, then the slide deck and finally practising my delivery.

A common mistake people make when preparing a presentation is to jump straight into the slide deck and attempt to develop the content whilst creating the slides.  This is ok if you're only speaking for five minutes, but when you're speaking for an hour you need to ensure that you are telling a compelling story with a powerful message.  If you don't work out the story and the message first, your audience will find it harder to engage with your presentation and likely switch off.

I started with a blank sheet of paper and a pen.  My colleagues mocked me for going analogue, but I wanted to be completely removed from PowerPoint, so I had no temptation to start creating slides until I had a compelling story to tell.

<img src="{{ site.github.url }}/images/2018-07-09/onion.jpg" style="width: 15em; float: right;" alt="The Onion Model">
Fortunately, I had some help from [Scott Solder](https://www.scottsolder.com/), an excellent speaking coach that Auto Trader works with from time to time.  He introduced me to a framework he'd developed called the writer's onion, which uses the [rule of three](https://en.wikipedia.org/wiki/Rule_of_three_(writing)) to give your writing structure.

The idea being that your presentation will start with an introduction and end with a summary, but the meat in the middle is broken down into three distinct sections.  Each section can then be broken down into three smaller sections if needed.  In the introduction, you state the three things you're going to talk about and in the summary, you remind your audience of the three things you have just talked about.  

The final piece of the puzzle is a one-liner with the key message you want your audience to take away.  In my case, this was:

> Continuous Improvement is a journey, and each step along the way reveals something new on the horizon.  Anything can be improved upon, any challenge can be overcome.

So I started to fill out my onion by writing a single word in each box until I had the basic structure defined.  I then took each of those words and brainstormed the finer detail of what I wanted to say about them.

Only then was I ready to start creating the slides.  Which was much easier to do, now I had the story prepared.  I simply started at the beginning and worked my way through to the end.  

The truly hard part was practising the delivery.  Slowly tweaking what I said on each slide to ensure that I wasn't repeating myself and that the talk was the right length.  I had to start this well in advance of the conference, because the slide deck needed to be submitted in December, two months before the conference.  If changes to the slides were needed, that was my deadline.

<img src="{{ site.github.url }}/images/2018-07-09/on-stage.jpg" style="width: 30em; float: left; padding-right: 1em;" alt="Matt Beaney on stage at Pink18">
By the time I got on the plane, I was tired of repeating the same words ad nauseam, but when I actually stood on the stage in front of the audience I was glad I'd put the effort in.

The final stage of preparation was to work on my delivery style.  Again, I had help from Scott for this.  We worked on our stances and practised different ways of getting back on track if something went wrong.

If you're planning on speaking at a conference, I really can't stress enough how invaluable a good speaking coach can be.

## Enterprise Continuous Deployment - Is it feasible...is it desirable?
The talk I gave was about Continuous Deployment: its benefits, its challenges and its relevance to the enterprise.  The session is available in its entirety here:

<iframe width="100%" height="315" src="https://www.youtube.com/embed/fEKZpo_to7k" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

### Definition
I define Continuous Deployment as:
> Continuous Deployment is an automated release process, where every build is deployed into production with no human intervention, provided that all automated tests have passed and the environment is healthy.

Most definitions of Continuous Deployment only state that the automated tests need to have passed.  I believe that simply checking that the code to be deployed is "safe" isn't enough.  It is equally important to know that the production environment is in a healthy state before releasing new code to it.

<img src="{{ site.github.url }}/images/2018-07-09/CI-CD-CDp.jpg" style="width: 60em; float: center;" alt="Continuous Integration, Continuous Delivery and Continuous Deployment in context">

This slide shows Continuous Deployment in context with Continuous Integration and Continuous Delivery. I used it to ensure that my audience fully grasped the difference between these three methodologies.

What's important to understand, is that progressing from Continuous Integration to Continuous Delivery & beyond requires more than simply automating the tasks along the top of the diagram.  You could conceivably practise waterfall delivery with automated tests & deployments.  

As you make this progression, what you're really doing is automating the _decisions_ between the tasks.  

To enable Continuous Delivery, where you _aim_ to release every green build into production, you first need to automate the decision of which builds get promoted into your test environment.  Any build that passes the initial tests will automatically be deployed to the test environment.  The goal here is to use the pipeline like a filter, with only minimal manual testing just prior to the production release.

To enable Continuous Deployment, you automate the final decision: the selection of the release candidate itself.

### The Benefits of using Continuous Deployment
Continuous Deployment, like Continuous Delivery & Continuous Integration, is a tool that you can use to improve your release success rate by making releases smaller and more frequent.

<img src="{{ site.github.url }}/images/2018-07-09/stats.jpg" style="width: 60em; float: center;" alt="smaller more frequent releases really work">

I demonstrated the power of this concept with the slide above, which shows how our release success rate has improved in line with the increase in our release volume.  The truly telling point on this slide is when you look at how the number of customer impacting releases has fallen at a similar rate.

Each FY on the graphs is a Financial Year ending in March.  E.g. FY17 is April 2016 to March 2017.  

I gave this talk in February 2018, so was unable to include the latest figures, but in FY18 we delivered 5044 releases with only 32 customer impacts and I'm currently projecting 8,500+ releases with around 20 customer impacts for FY19.

The step change between FY16 & FY17 on the first graph shows where we enabled our product squads to use Continuous Delivery.  The other step change you can see on the final graph from FY14 to FY15 is another story, which I've explained [in a previous post on this blog]({{ site.github.url }}{% post_url 2017-09-15-it-matters-what-you-measure %}). 

In addition to driving smaller more frequent releases, Continuous Deployment enforces good practice like the use of TDD, Feature Shields, Health Checks & Automation.  I covered each of these in some detail in the talk, but of particular note are Feature Shields and Health Checks.

[Feature Shields or Feature Toggles](https://martinfowler.com/articles/feature-toggles.html) are used to prevent unfinished code from blocking releases, by disabling that code in production until it's ready.  This is critical if you want to even get close to Continuous Deployment, in fact, I'd say that Feature Shields are so useful they should be considered even if you're only just getting started with Continuous Integration.

Health Checks are a really powerful tool to not only improve your release success rate but also speed up incident diagnosis.  There are two kinds of checks that are relevant to Continuous Deployment.  I've already mentioned how important it is to have an environment health check.  In my talk, I explain how this is actually more of a kill switch, by the time you're using Continuous Deployment.

<img src="{{ site.github.url }}/images/2018-07-09/dozor.jpg" style="width: 15em; float: right;" alt="Application health checks in action">
The other health checks of note are application health checks, which are used by the automated deployment to confirm that a release has started cleanly on a node before routing any traffic to it.  This is a basic safety-net which you will quickly take for granted, but you can't deploy safely without.

To aid incident diagnosis, we have built an app called Dozor which continuously polls all of our applications health checks to confirm that all nodes are up and healthy.  This helps us pinpoint not only which of our 250+ applications is struggling, but also whether it is one node in 20 or the entire pool.

### The Challenges of using Continuous Deployment
The challenges you will face if you try to implement Continuous Deployment are many & varied.  Moreover, there will likely be challenges that are unique to your business.

The biggest challenge, the hardest to overcome and the one that *everyone* has to face is Fear or [FUD.](https://en.wikipedia.org/wiki/Fear,_uncertainty_and_doubt)  **Fear is the mind-killer.**

The reason Fear is such a problem is that it's hard to trust a machine to make decisions that have always been made by a human being previously.  No matter how many graphs I show you, the idea that these decisions are actually safer in the hands of a machine is counter-intuitive.

The best way to overcome Fear is to build confidence slowly, through continuous improvement.  Bear in mind that nothing happens in isolation.  It takes a lot of time to put all the pieces in place that are required to Continuously Deploy safely.  Use this time to really nail Continuous Delivery, build confidence in your test coverage and assess which of your applications are ready.

- Think about the complexity of the application, how easy is it to get a definitive test result?
- Think about the potential impact if it all goes horribly wrong: is it critical?  
Eventually---as you gain confidence---this won't be a blocker. In the meantime, you will want to cherry-pick apps you don't care about all that much.
- Think about the benefit of enabling Continuous Deployment for the specific application you're evaluating.
The first customer-facing application we enabled was our partner search.  This is used by a variety of third-parties to embed our search form in their page.  We picked this application because it currently only gets released a couple of times a year, however, it includes our core search library which is in constant development.  This means that when we do release it, it's pretty scary because it includes a massive amount of change.  By enabling Continuous Deployment we have made this much safer because it is now released when anything changes, whether it is a direct change for partner search or not.

In addition to this, you need to consider the complexity of your architecture, the size of your development team(s) and your current release success rate.  All of these factors will affect your ability to safely deploy and may provide sources of Fear.

Finally, you should assess where the Fear originates.  If your technical staff are concerned, there is likely something that needs to be addressed and you should discuss their concerns with them to ensure nothing has been overlooked.  However, if your tech teams are confident, but your management layer is not, you can have a very different conversation.  Sell them on the benefits of smaller, more frequent releases.  Explain the controls that need to be put in place to protect the business.  Use their Fear to encourage them to allocate the resources required to do this safely.

### Is Enterprise Continuous Deployment either Feasible or Desirable?
<img src="{{ site.github.url }}/images/2018-07-09/enterprise.jpg" style="width: 30em; float: left; padding-right: 1em;" alt="you CAN do it">
Most of the businesses using Continuous Deployment are either small startups or tech giants like Google & Amazon.  In my presentation, I highlight some of the differences between startups and enterprises.  I talk about the advantages that a small, nimble business has, but also that almost all of those advantages are double-edged swords.

Continuous Deployment is harder to achieve in an established enterprise, but harder is not the same as impossible.  The real difficulty lies in aligning the entire organisation towards it.  A strong DevOps culture is absolutely essential, without it you will never get past the Fear barrier.

Used properly, Continuous Deployment is a tool that can safely speed up your release cycle.  It is not, however, something that you *need* to use globally.  At Auto Trader, we have a small number of applications that will never progress past Continuous Integration.  They will likely be phased out over time and until then we'll simply continue to release them as they are.

This is not a problem.  

One-size-fits-all solutions are a myth.  In reality, you will always have edge-cases.  Some of which you may be able to cater for, but in trying to cover them all you will only compromise your solution for the bulk of applications.

Identifying those applications which are not eligible for any new deployment method is a critical part of a responsible roll-out strategy.

## The Experience
The last 20 minutes before I got on stage to deliver the talk were nerve-wracking, to say the least.  I had to keep reminding myself that everyone in the room had self-selected my session because they were interested in hearing what I had to say and that I knew both the material and the subject inside out.

As soon as I began to speak, my nerves began to settle and I found my rhythm very quickly.  I wasn't even phased when one of the few slides with any animation appeared all in one go, instead of allowing me to trigger each section to add emphasis to what I was saying.  There is a definite downside to presenting a copy of your presentation on someone else's laptop, which is, unfortunately, something Pink Elephant changed for Pink18.

I set out with a goal to inform and inspire my audience.  Seeing the reactions of people in the audience when a particular idea really resonated with them told me that I was succeeding.

The preparation had paid off and when I delivered my final line and stepped down from the stage, I was really buoyed up by the number of people who came forward to ask me questions.

The whole experience was very positive and speaking was definitely the highlight of the trip.  Although when I signed up for it I knew it was outside of my comfort-zone, I enjoyed it so much that I've already submitted a talk for next year!
