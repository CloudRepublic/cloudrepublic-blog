---
title: Following the moon
date: 2019-05-22 18:00:00
tags: 
- Azure DevOps
- Release schedule
- Release pipeline
---

In the ideal situation, we would like to release our new code with zero downtime. It is very possible that this can be done by releasing to a separate staging slot and then swapping this with the production slot. Should be zero downtime. However, there are always cases where this does not apply, where downtime can only be avoided through tedious manual intervention and multiple failover steps. Say we have an application that runs globally and we have such a case. Or if we just really, really want to make sure that our customers have as few problems as possible.

In this case, we propose to use the follow-the-moon (FTM) release schedule. This means releasing our application at times in different regions where for each region, the time we release at is the time where it is least likely that a customer is using the application. And yes, we know that the moon can be visible during the day, but you get the sentiment.

You will still want to have the entry point of your application to send your users to a region you host your application in that is 1) available and 2) close to the user. Taking a region down to release a new version of your application is still not desired, as it will either not be rerouted (e.g. because of caching) and thus will result in routing to an application that is down, or be rerouted to a region where the distance can cause undesirable increases in response times. Thus, we would like to make sure to do the release at a time where it is convenient per region, not all at once.

In the FTM release schedule, we may, for example, set the release time for each region to 03:00 local time. Once we approve the continuation of the release, the schedule will start to kick in and release our application to all regions at their respective optimal times. This means that our application is rolled out automatically, in different time zones, without the need for manual intervention in our multi-region roll-out process.

Getting our application to production globally could involve the following:
-	Set up a continuous integration build to automate building your code
-	Set up an automatic release pipeline, automatically started from artifacts, going through multiple stages with certain filters on source branches
-	Set up the follow-the-moon release schedule for our production releases to multiple regions

## Getting to the moon
For this part, we assume your project has some code and a build is in place to create artifacts which we can work with.

Suppose our release pipeline looks like this:

<img src="/images/follow-the-moon/follow-the-moon-1.jpg" />

We have our artifact as an entry point. We have our DEV and TST environments hooked up for continuous releases based on the develop branch. Finally, we have our ACC and PRD environments hooked up for continuous releases based on the master branch. In this case, we want to double-check the ACC environment before actually rolling out to PRD, so we add a post-deployment approval condition there.

Now, if we click on the pre-deployment conditions, we see the following menu:

<img src="/images/follow-the-moon/follow-the-moon-2.jpg" />

We enable the schedule and set it to the time where we expect our users to not use the application in that region. For example, in the WE (West Europe) Azure region, at 03:00 would be when we expect our customers to sleep, so we may decide that this is the right time to deploy.

<img src="/images/follow-the-moon/follow-the-moon-3.jpg" />

After we have done this for all the production environments, we have successfully implemented the FTM release schedule! Do note that using swap slots is still what you want to do. However, this principle gives us a little bit of extra safety when releasing code that may otherwise cause downtime.
