---
layout: post
show_meta: true
title: Using go-routines in AWS SDK V2
header: Using go-routines in AWS SDK V2
date: 2024-10-08 00:00:00
summary: Using go-routines in AWS SDK V2
categories: go-lang aws sdk
author: Chee Yeo
---

In a recent project which uses the AWS SDK V2 for go lang, I had to devise a strategy to wait for a single EC2 instance within an autoscaling group to be in a ready state in order to invoke a custom setup script. This script is meant to be run only once on setup and can only be run on the first available instance.


 ( TODO )