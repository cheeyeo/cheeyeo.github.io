---
layout: post
show_meta: true
title: AWS Aurora RDS external access
header: AWS Aurora RDS external access
date: 2024-12-07 00:00:00
summary: How to access Aurora RDS database externally using Network Load Balancer
categories: aws rds network-loadbalancer
author: Chee Yeo
---

[AWS Data blog post]: https://aws.amazon.com/blogs/database/connect-external-applications-to-an-amazon-rds-instance-using-amazon-rds-proxy/

One of the difficulties in using Aurora RDS from my own experiences, is the need to create a jumphost to connect to via SSM and from the CLI, test the ability to connect to it. Most RDS databases are provisioned within a database subnet in a private VPC, which means it cannot be accessed publicly for security reasons.

In a [AWS Data blog post], a process was described whereby we could create an RDS proxy for the database and set the Proxy's vpc endpoint IP as network load balancer listeners to be able to access the RDS instance externally. In this post, I attempt to describe the process I re-created the steps highlighted in the original blog post.

I used terraform with terragrunt to create the resources. The benefit of using terragrunt is that it makes it easier to separate the resources into its respective statefile, which means, we can turn off external access to the database by deleting the network load balancer and proxy when not required.