---
layout: post
title:  "Postmortem on the application outage on Fri 26th May"
author: Dave Whyte
---
On Friday 26th May, we experienced a major service outage for 75 minutes around one of our core products, VDS (Vehicle Data System). The result of this was some applications not being able to return vehicle-specific data.

The outage occurred during a planned network maintenance window, the main cause of the outage was due to the VDS endpoint not being switched to our standby Data center (DC) before the maintenance.

Leading up to the outage, Engineering had identified an issue with Stats collection on our Local Traffic Managers (LTM's). These are at the heart of our architecture, load-balancing connections to our application servers integral to our services. After an investigation, we agreed that a reboot was the final course of action to resolve the problem. 

The LTM's in question operate in an HA (High-Availability) pair so a rolling reboot without service disruption should work fine. However, as we are in the fortunate position of running two fully functional DC's we took the decision to move all traffic away from the LTM's in question and run in DC2 while we performed the maintenance required in DC1.
We manipulate traffic inbound to our Data Centers via DNS. Essentially if we want to change the DC, a service is running from, we alter the DNS record to present an IP address from that DC. We can do this for one service, multiple services or all services, to make a change we use an in-house tool called TRS. TRS is an in-house front end interface to the F5 GTM (Global Traffic Manager).

The pre-agreed maintenance actions:

1.            Use TRS to point ALL services to DC2
2.            Reboot Standby LTM in DC1
3.            Once rebooted, failover to Standby
4.            Test
5.            Reboot previously Active LTM
6.            Once rebooted, place LTM's back in default position (i.e., put Active LTM back in service)
7.            Test
8.            Use TRS to point ALL services back to default position (DC1)

What happened:

1.            Use TRS to point ALL services to DC2.
At this point, without knowing, we suffered our first failure. TRS did not work as expected, not ALL services switched to DC2, and the VDS endpoint remained in DC1. We were blind to this failure as we did not verify visually that all services had switched.
2.            Reboot Standby
3.            Once rebooted failover to Standby
4.            Test
Currently, everything is still happy from our point of view, going to plan.
5.            Reboot formerly active LTM.
Here we walk into failure number 2. The formerly Active LTM (now standby) rebooted and came up Active. Both LTM's were now Active, creating a split-brain problem between the LTMs rendering the LTM's inoperable.
Our monitoring systems now began to identify services affected, both internally and externally. 

We still believed that all services had switched to DC2, our internal and external monitoring was alerting us to issues with a few core services, the symptoms were slowness and a build-up of connections.  At this point the issues were investigated as a possible separate issue not connected to the LTM maintenance, we restarted some of the affected systems, but this did not resolve the issue, and the impact of the issue was getting worse over time with more systems starting to be affected.
Utilising one of our internal monitoring tools (Dozor), we could identify, via a service health check, that there appeared to be an issue connecting to the VDS endpoint. With this knowledge, we quickly identified that the endpoint had not switched to DC2 when we made the earlier change. Upon this discovery, we switched the service to DC2, and after a short time, all the alerting started to clear, confirming that this was the reason for the system issues.
Now that all services were working in DC2, we continued to investigate and resolve the issue with the LTM's in DC1 without any further service impact.

As standard practice following any major service outage, we held a review utilising a timeline of the incident to highlight key events. We then discussed what happened in more detail (using What, How and Where questions) along the way to identify key learnings and identify immediate actions to improve our stability

Agreed actions:

1.            Arrange the rewrite of TRS; we were already planning to rewrite this application, and when we do, any lessons learned from this incident should be incorporated.
2.            For now, ensure the support teams are aware that they need to carry out a visual check when switching large groups of services.
3.            Liaise with SecData (LTM third party support) to identify the cause of both LTM’s becoming active at the same time.  
4.            Test the failover of the LTM's again to capture more information as we no longer have the logs.
5.            During any incidents, ensure the support team is liaising with the relevant product squads.
6.            If known, communicate a more technical explanation of the cause as soon as possible on Slack.
7.            Ensure that we update our status.io page (http://atstatus.co.uk/) if we are seeing any customer impact.
8.            Write our first post-major service outage blog.

It is worth pointing out that incidents of this nature are few and far between, the impact was not widespread, and it is reassuring that it took a chain of failures (software and process) before any service impact incurred. However, it is important we learn from events like this. Completing the agreed actions should help with ensuring we do not see an occurrence of this outage with the same cause again. One of the themes in the review was around internal communication. The product squad responsible for VDS felt out of the loop of the incident and only really understood what happened when discussing it at the review. Completing the actions should help with improving the overall communication during an outage.
