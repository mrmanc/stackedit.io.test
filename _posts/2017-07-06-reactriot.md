---
layout: post
title:  "ReactRiot!"
author: Nick Baker
tags: [Hackathon, React]
---

On the weekend 24-25<sup>th</sup> June, three developers from Auto Trader (Leon Pelech, David Carter and myself) participated in the first [ReactRiot](https://www.reactriot.com/) hackathon. It was a two-day event with teams competing from all over the globe.  

<img style="margin-left: 35%; width: 30%;" src="{{ site.github.url }}/images/2017-07-06/reactriot-logo.jpg">

# The Rules
The basic rules were:
- Applications must be built using React.
- You have 48 hours, from midnight on June 23<sup>rd</sup> to 23:59 (UTC) on June 25<sup>th</sup>. No code can be written before this time.
- Teams of 1 - 4 people 
- Winners will be picked based on a mixture of votes from judges, other participants, and the public.

(Official rules can be found [here](https://www.reactriot.com/rules))

# The Idea

<img style="float: right; width: 20%; padding-left: 10px;" src="{{ site.github.url }}/images/2017-07-06/reactvr-logo.png">

When coming up with ideas, we very quickly gravitated towards something using [React-VR](https://facebook.github.io/react-vr/) as we had all seen it on our trip to the [React Amsterdam](https://react.amsterdam/)
conference. Some of the concepts being thrown around were:
- A virtual reality development environment
- A VR shop
- A VR Manchester journey planner

To be honest, take any website you can think of, put 'VR' in front of it, and we probably considered it as a concept.

Eventually, we had the idea of using the stats produced by a [webpack](https://webpack.github.io/) build. We use webpack for our React projects at Auto Trader, and it is a common way to bundle your code in the React community. 

The initial concept was to make some kind of game based on the stats. We wanted to generate a dungeon that the user could move through, with monsters based on the contents of the bundles. When we sat down to start planning this, we quickly realised that this was perhaps a little ambitious for a two day hack, and decided to limit the scope to a virtual reality visualisation of the statistics.

The requirements we came up with were:
- The user can upload their webpack build statistics
- The webpack chunks are displayed as some sort of three-dimensional object
- Examining a chunk gives the user more information about it

Most importantly, we needed a good name for the project. A few ideas were discussed, but eventually we settled on *Bundle Dungeon*, which was the working title of our game idea.

# The Team
## Carter, David
- **Role on team:** The ideas man

Carter came up with many of the ideas for this project and did the lion's share of the front end work. Without his vision of how the product would end up, we'd never have managed as much as we did.

## Pelech, Leon
- **Role on team:** The hacker

Without a doubt, it was Leon's ability and knowledge of React that brought this hack together. He did the majority of the setup for the project, and is still the only one of us who actually knows how to deploy the thing! Even though he ended up spending a bit too long chasing some dead ends, he carried on hacking away without complaining.

## Baker, Nick
- **Role on team:** The number cruncher

I ended up getting lumped with the maths. It turns out that positioning objects semi-randomly in three dimensional space, making them rotate, and then getting stuff to appear near them gets complicated. I spent a good few hours staring at trigonometry and calculus to figure out how on earth we were going to do it.

# The Hack
We were allowed to use the Auto Trader office for the weekend. It was strange at first to see the office so dead and empty, but it probably let us work more efficiently than our backup location would have done.

We met up at about 8:30 on Saturday morning, grabbed breakfast and began discussing plans. Leon set up the skeleton project whilst Carter and I read the React-VR documentation. 

The tasks were divided up such that Leon was working on the file uploader (for importing webpack statistics), Carter was making some blobs appear in virtual space, and I was doing maths.

Unfortunately, the file uploader that Leon claimed would be a "five-minute job, ten max" turned into a 5-hour epic, involving him doing all sorts of crazy hacks to try and integrate it into a React-VR environment. This became a recurring theme for our day, and it felt like everything we wanted to include ended up becoming a monster task due to the way React-VR is structured (or our lack of understanding of it).

After day one, we had something resembling what we wanted, but it felt like we'd hit the limits of the framework. We were a little deflated, and thought that we'd end up spending all of the second day trying to figure out how to add any features to the giant mess of code we'd written.

<img style="margin-left: 25%; width: 50%;" src="{{ site.github.url }}/images/2017-07-06/autotrader-campervan.jpg">

Day two began with a meeting in a campervan (one of the quirkier meeting rooms at the Auto Trader offices), debating whether we should scrap 90% of what we'd done and start again with [aframe](https://aframe.io/), another VR library that seemed a little more flexible than React-VR. After a long chat, we figured that we'd actually achieved so little on day one that we might as well give aframe a go to see how far we could get.

<img style="float: left; width: 20%; padding-right:10px;" src="{{ site.github.url }}/images/2017-07-06/aframe-logo.png">

Within a few hours, the aframe version of our app had all of the features we'd written in React-VR. I was happily doing trigonometry again, and the whole team was reinvigorated. We carried on adding all of the features you can see in the app today.

At about 11 p.m. on the Sunday evening we called it a day. There were no more feature ideas we could realistically build in under an hour, and we were all shattered! Feeling very proud of what we'd built in a day, we all headed home.

# The Finished Product
You can experience Bundle Dungeon yourself [here](https://hackbit.github.io/reactriot2017-team-at/)! If you have access to a Google Cardboard headset, I recommend using that for the full VR experience. 

<img style="margin-left: 25%; width: 50%;" src="{{ site.github.url }}/images/2017-07-06/bundle-dungeon.png">

For instructions on how to use the app, take a look at our [team page](http://www.reactriot.com/entries/228-team-auto-trader-uk) on the ReactRiot website.

# The Competition
We received some wonderful feedback about our project from both judges and other participants. Everyone seemed to love the idea of playing with the stats we see every day. One complaint we got was about some performance issues (something we had observed), but we can live with that given that it's a two day hack project.

We did not end up winning any prizes, but we had so much fun over the weekend that we are intending to find more hacks to compete in. Hopefully the more time we spend as a team, the more productively we can work together. 

## We weren't the only entry!
All of the projects for ReactRiot are now available online [here](http://www.reactriot.com/entries/all), and I would recommend checking a few out. Some of them are phenomenal! My personal favourites are:
- [Composer](http://www.reactriot.com/entries/331-teamninja) A musical sequencer. (This won the "Hacker's favourite" Category)
- [Hana](http://www.reactriot.com/entries/233-sans-team) An odd-jobs classified platform.
- [PewPew](http://www.reactriot.com/entries/257-lyanglyang) A multiplayer game where you sing to shoot at your friends. (This won the "Utility/fun" category)

# Learnings
The main things we learned were:
- **Don't be afraid to pivot:** Switching to aframe pretty much saved the project. Even with half the time, we almost certainly had a better finished product than if we'd persevered with React-VR
- **VR is hard:** For developers used to working in the 2D environment of a webpage, developing a product in 3D added many layers of complexity.  
- **We really like doing hacks:** As mentioned above, we had a great time doing this. We had very little hackathon experience between us going in, but this has made us want to participate in more.

I had an awesome time as part of Team Auto Trader in ReactRiot. Whilst we didn't win anything, I am still really proud of what we built. Virtual Reality is an incredibly exciting area of software development right now, and dipping my toes into it for the first time was interesting and rewarding.
