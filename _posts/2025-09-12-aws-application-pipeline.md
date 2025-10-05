---
layout: post
show_meta: true
title: Implementing the AWS Application Pipeline in Github Actions
header: Implementing the AWS Application Pipeline in Github Actions
date: 2025-09-12 00:00:00
summary: An exercise in deploying the AWS Application Pipeline using Github Actions
categories: aws devops ci cd pipeline
author: Chee Yeo
---

[AWS Deployment Pipeline Reference Architecture]: https://aws-samples.github.io/aws-deployment-pipeline-reference-architecture/application-pipeline/


This post shows how I implemented the pipeline detailed in [AWS Deployment Pipeline Reference Architecture]. The guide specifies 2 different types of pipelines:

* Application Pipeline, which is the standard approach to developing and publishing software using CI/CD approaches.

* Dynamic Configuration Pipeline, which is an approach to manage configuration changes for multiple microservices so that all the configuratins for each of the microservice are tracked and allows for code review process such as pull requests and reviews

This post discusses the Application Pipeline


