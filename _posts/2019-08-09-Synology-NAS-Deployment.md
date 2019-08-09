---
layout: post
title: Synology NAS Deployment
description: "Deploying a NAS for a small business is easier than you think."
author: josh
category: storage
tags: storage networking data
finished: true
---

A client reached out recently requesting to upgrade their storage solution for their office. They have roughly 12 users and they're expanding to a second office in another state with a handfull of users. They're currently using a single computer running windows 7 using an internal hard drive as a network share for their users. The remote office uses LogMeIn to a machine in the main office in order to access the drive. They aren't interested in setting up a full domain yet and AWS solutions are too pricey per month for the amount of data they're accessing.

# In comes Synology

Recommended Gear:

- 2 x Synology DS218+ NAS Bays
- 4 x 6TB WD Red drives

This provides the team with a NAS and 1 level of redundancy at each location. Farewell LogMeIn!

## Setup

Synology makes the setup fairly easy but there are a handful of controls that can make things better or worse.