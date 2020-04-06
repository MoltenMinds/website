---
title: Issues when using configuration files in Scaling environments
date: '2020-03-09'
thumb_img_path: images/scale.png
content_img_path: images/scale.png
excerpt: >-
  Dealing with issues on Azure Blob Storage
layout: post
author: dani
---

Hello,

We will continue our last post on problems that appear when scaling. This time we want to talk about what happens when you need to use configuration parameters in an environment. These values can be stored in many different places. In the case of the project we had, they were stored in <a href="https://azure.microsoft.com/en-us/services/storage/blobs/" > **Azure Blob Storage** <a/>. The problem in this case appeared when enabling auto scaling.

## About the project

First we need to explain the project. It was a complex backend application in **C#**. The architecture of the project was design with scaling in mind. It consisted of Frontend and Backend App Service Servers. Each server contained many Webjobs. The different Webjobs needed several parameters to run correctly. When they were lunched each one needed to read a configuration file containing these parameters.

We decided to use configuration files instead of saving the values directly to the database, first because a performance issue but also to be able to have different sets of parameters for each region and environment. These values could be changed from the frontend so each job had to refresh them after some time so the changes propagate across the app in a similar way the DNS changes propagate.

To read the files whe had a three checks, first we search in local memory cache for the file. Then we search in  <a href="https://azure.microsoft.com/en-us/services/storage/blobs/" > **Redis** <a/> if not present we look for the file in  <a href="https://azure.microsoft.com/en-us/services/storage/blobs/" > **Azure Blob Storage** <a/>.

With this architecture the program already was running in multiple regions with multiple servers. And in each region it was configured with App Service using a considerable number of instances.

## The Problem

The Application was working as expected. And a new objective was defined to reduce the costs. After solving the scaling problems we started to find a couple times during two weeks periods where the program was logging errors find errors when accessing the configuration files.

At first we thought there was a problem with Azure Redis and Blob Storage both at the same time but the issue happen again after some time and that couldn't be a coincidence. Then we started to investigate the possible cause of the issue.

## The solution

After a days of reading what could cause this reading errors when accessing the configuration files. The first thing we could correlate was our errors with the down times of Redis cache service. So we now knew that each server could have at most 20 instances, these instances had more 30 jobs so we have at most 600 process running a request asking for the file. That is if every instance was started at the same time. So why would the Blob Storage fail for 600 file requests?

We found a <a href="https://docs.microsoft.com/en-us/azure/storage/blobs/scalability-targets" > **dcoumentation** <a/>  on scalability and performance for Blob storage. There we can see that there is throttle limit independent of the Storage tier you have. This limit is 500 request per second, that we are hitting sometimes, even with our retry policy we are having errors.

The retry policy was waiting for a constants time, upon changing it for an exponential one most of the issues were resolved.

Then we read about the <a href="https://docs.microsoft.com/en-us/azure/storage/blobs/storage-performance-checklist#partitioning" > **naming convention** <a/> to improve Storage Service load balncing. We were saving in the same Storage account all the queue messages leasing files without appropriate naming. Upon changing that we could resolve all the issues with the blob services.

Hope you learnt something about managing Blob files in a scaling application.

Regards.