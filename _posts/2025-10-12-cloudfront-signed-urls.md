---
layout: post
show_meta: true
title: Cloudfront signed urls
header: Cloudfront signed urls
date: 2025-10-12 00:00:00
summary: Cloudfront signed urls
categories: python aws cloudfront
author: Chee Yeo
---



Associate CF distribution with S3 Bucket but distribution is public which is accessible over public internet

To secure distribution, CF allows you to attach WAF but its not ideal if your application is meant to be public....

Use signed urls or signed cookies to restrict access based on RSA key pair

use openssl to create private-public key pair
public key uploaded to CF via `Public Keys`
create `Trusted Key Groups` using the uploaded public key
Modify the distribution behaviour and change it to `Restrict viewer access` and select the `Trusted Key group` created above ^

( show screenshots )

If user tries to access the original distr url, it will fail with error message of missing private - public key pair`

Need to use boto3 for example to generate signed url

( show boto3 script )

using signed url allows you to access the restricted content; url only valid up till the time period specified by the signer...


### how it works ?? 

CF uses public key from trusted key group to create a signed signature of the URL

User uses boto3 program and private key to generate signed url for access

CF compares the generated signed digest to the one it generates using the publi key and if it matches access is aallowed...