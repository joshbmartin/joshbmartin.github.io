---
layout: post
title: Synology NAS Deployment
description: "Deploying a NAS for a small business is easier than you think."
author: josh
category: storage
tags: storage networking data
finished: true
---

A client reached out recently requesting to upgrade their storage solution for their office. They have roughly 12 users and they're expanding to a second office in another state with a handfull of users. Currently an internal HDD on a Windows 7 machine is being utilized as a network drive for the main office (Office A). The remote office (Office B) uses LogMeIn to access a machine in Office A in order to access the drive. They aren't interested in setting up a full domain yet and AWS solutions are too pricey per month for the amount of data they're accessing.

# In comes Synology

Recommended Gear:

- 2 x Synology DS218+ NAS Bays
- 4 x 6TB WD Red drives

This provides the team with a NAS and 1 level of redundancy at each location. Farewell LogMeIn!

## Setup

Synology makes the setup fairly easy but there are a handful of controls that can make things better or worse. Thankfully they make it easy to sync up multiple Synology drives by installing Cloud Station Server to the main NAS and Cloud Station Client to the secondary NAS. 