---
layout: post
show_meta: true
title: Building Flask web application in Github Actions
header: Building Flask web application in Github Actions
date: 2025-08-29 00:00:00
summary: How to build a Flask web application using Github Actions
categories: python github-actions ci/cd flask
author: Chee Yeo
---

In my previous post, I described a process of using inline services block to run an external Postgresql database to run unit tests as part of the CI pipeline for a Flask web application.

However, running unit tests is just a single aspect of a CI/CD pipeline. As documented by the AWS well-architected pipeline for CI/CD workflows, we also need to include additional stages such as code linting and formatting; secrets detection; and static security testing. This post will detail how I incorporated those additional stages into the Github pipeline before building and pushing the final docker image into ECR.




( TODO )