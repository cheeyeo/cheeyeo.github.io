---
layout: post
show_meta: true
title: Cloudfront signed urls and cookies
header: Cloudfront signed urls and cookies
date: 2025-10-10 00:00:00
summary: How to use Cloudfront signed urls and cookies
categories: python aws cloudfront
author: Chee Yeo
---

[Restrict content with signed URLs and signed cookies]: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PrivateContent.html

[Creating a signed url]: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-creating-signed-url-custom-policy.html

In web projects, it is common practice to upload UI assets into S3 and associate a Cloudfront distribution with it to speed up delivery of assets. Generally, these CDNs are public accessible over the public internet. 

However, there might be instances whereby certain files are restricted but you still want to make use of Cloudfront. By default, under the Security tab for each distribution, you can restrict access via Geographic locations or even enable AWS WAF ( Web Application Firewall ). But to use WAF, there are additional charges for each request made to the distribution and may not be practical for a web application. In addition, the security is applied to the entire distribution rather than individual files.

To restrict access based on individual files, we can [Restrict content with signed URLs and signed cookies].

Firstly, we need to use `openssl` to create a private-public key pair. The private key must be RSA 2048 or ECDSA 256 compatible. I ran the following command below to create a test keypair for this post:

```
openssl genrsa -out private_key.pem 2048
```

To retrieve the private key, we run the following command:
```
openssl rsa -pubout -in private_key.pem -out public_key.pem
```

Next, we upload the public key to `Public Keys` under Cloudfront.

Once completed, we create a `Trusted Key Groups` in CloudFront and reference the uploaded public key.

Next, we modify the distribution behaviour and change it to `Restrict viewer access` and select the `Trusted key group` created previously. Below is a screenshot that shows the process:

The above is known as `associating a signer with the distribution`. Each signer ( keypair ) is associated with the cache behaviour of the distribution. It's possible for a distribution to have multiple behaviours and some of them have a signer associated. 

If we try to access the public url of the Cloudfront distribution for that specific file, it will fail with error message of missing private - public key pair. The distribution is now looking for a `Signature` query parameter. Cloudfront will block access if this is missing. 


The example below uses the `cryptography` and `boto3` libraries to generate a signed url using the `CloudFrontSigner` class in boto3

( show boto3 script )
( gist )











### how it works ?? 

In CF distribution, when you associate a Trusted Key Group, CF uses the public key associated with it to verify the URL signature. 

Your application uses the private key to generate the URL signature.

Within your application, you set restrictions on the conditions to access the files i.e. based on whether the user is logged in or not.

After user access is granted in your application, it uses something like `boto3` to generate the signature using the private key and returns it to the user.

The user access the signed url using a browser for example

CF receives the signed url and verifies the signature using the public key; if is not tampered with, it then checks the expiry date and any custom policy attached to ensure its still valid.

Then access is allowed.

Cloudfront checks the edge cache location to determine if the file is available; if not it forwards the request to the origin and returns the file to the user while also updating the edge cache location ( write-through cache )
