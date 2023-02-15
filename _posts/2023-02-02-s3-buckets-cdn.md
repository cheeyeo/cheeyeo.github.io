---
layout: post
show_meta: true
title: Using CloudFront Distribution with S3
header: How to setup a CDN with private S3 bucket
date: 2023-02-02 00:00:00
summary: How to setup a CDN with private S3 bucket
categories: terraform aws s3 cloudfront devops
author: Chee Yeo
---

[Origin Access Control]: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html#create-oac-overview

[Supported Compression FileTypes]: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/ServingCompressedFiles.html#compressed-content-cloudfront-file-types

[Private Content Task List]: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-task-list.html

[Optimizing caching and availability]: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/ConfiguringCaching.html


In a recent project, I started looking into how CloudFront distributions work in relation to serving static content stored in S3 buckets. 

In this article I aim to explain how to setup a cloudfront CDN with an existing private S3 bucket with compression.

To allow access between cloudfront and the private S3 bucket we need to:

* Create an [Origin Access Control] (OAC) policy to allow cloudfront to send authenticated requests to S3.

* Create a bucket policy for the S3 origin to only allow access to it from that specific cloudfront distribution.

To create the OAC in terraform, we can use the `aws_cloudfront_origin_access_control` resource:

{% highlight terraform %}
resource "aws_cloudfront_origin_access_control" "default" {
  name                              = "S3Assets"
  description                       = "S3 Assets CDN Policy"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}
{% endhighlight %}

For the bucket policy we can create a `aws_iam_policy_document` resource:

{% highlight terraform %}
data "aws_iam_policy_document" "allow_access_from_another_account" {
  statement {
    sid    = "AllowCloudFrontServicePrincipalReadOnly"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["cloudfront.amazonaws.com"]
    }

    actions = [
      "s3:GetObject",
    ]

    resources = [
      "${data.aws_s3_bucket.selected.arn}/*",
    ]

    condition {
      test     = "StringEquals"
      variable = "AWS:SourceArn"
      values   = [aws_cloudfront_distribution.s3_distribution.arn]
    }
  }
}
{% endhighlight %}

The policy only allows access to the contents of the private s3 bucket from cloudfront if its source arn of the requester matches that of the distribution. 

Assuming we have a data resource of the bucket we can attach to it through a `aws_s3_bucket_policy` resource:

{% highlight terraform %}
data "aws_s3_bucket" "selected" {
  bucket = var.aws_s3_bucket
}

resource "aws_s3_bucket_policy" "allow_access_from_another_account" {
  bucket = data.aws_s3_bucket.selected.id
  policy = data.aws_iam_policy_document.allow_access_from_another_account.json
}
{% endhighlight %}


Next we can create the cloudfront distribution using the following:
{% highlight terraform %}
data "aws_s3_bucket" "selected" {
  bucket = var.aws_s3_bucket
}

# Gets the AWS managed cache policy
data "aws_cloudfront_cache_policy" "caching_optimized" {
  count = length(var.prebuilt_policy_name) > 0 ? 1 : 0
  name = var.prebuilt_policy_name
}

resource "aws_cloudfront_distribution" "s3_distribution" {
  origin {
    domain_name              = data.aws_s3_bucket.selected.bucket_regional_domain_name
    origin_access_control_id = aws_cloudfront_origin_access_control.default.id
    origin_id                = var.origin_name
  }

  enabled = var.enable_cdn

  logging_config {
    include_cookies = false
    bucket          = "${var.aws_s3_log_bucket}.s3.amazonaws.com"
    prefix          = var.aws_s3_log_prefix
  }

  default_cache_behavior {
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = var.origin_name
    viewer_protocol_policy = "redirect-to-https"
    compress               = true

    cache_policy_id = length(var.prebuilt_policy_name) > 0 ? data.aws_cloudfront_cache_policy.caching_optimized[0].id : aws_cloudfront_cache_policy.compression[0].id
  }

  restrictions {
    geo_restriction {
      restriction_type = "whitelist"
      locations        = ["US", "CA", "GB", "DE"]
    }
  }

  tags = {
    Environment = "dev"
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
{% endhighlight %}

In the `origin` block we define:
* the domain name to the regional domain name of the s3 bucket
* sets the origin access control (OAC) to reference the one we created earlier
* provide an origin name as a reference

We declare a `logging` config that stores the access logs of the distribution in a separately created bucket with a prefix value.

Next we define the cache behaviour. Note that:

* we set the compress option to true. This is required as its off by default and no compression is applied otherwise.

* We define a separate cache policy id as the source of this behaviour. We are using the default system defined `Managed-CachingOptimized` policy which enables both gzip and brotli compression but doesn't set any query, headers, cookies. 

Note that we also define the option of allowing the user to either use the default policy or to use a custom policy. 

If we are defining a custom user policy to enable compression, we could declare it using `aws_cloudfront_cache_policy` resource like so:

{% highlight terraform linenos %}
# Compression cache policy
resource "aws_cloudfront_cache_policy" "compression" {
  count = length(var.prebuilt_policy_name) > 0 ? 0 : 1
  name        = "compression-policy"
  comment     = "Compresses assets from origin"
  default_ttl = 86400
  max_ttl     = 31536000
  min_ttl     = 1

  parameters_in_cache_key_and_forwarded_to_origin {
    cookies_config {
      cookie_behavior = "none"
      cookies {
        items = []
      }
    }

    headers_config {
      header_behavior = "none"
      headers {
        items = []
      }
    }

    query_strings_config {
      query_string_behavior = "none"
      query_strings {
        items = []
      }
    }

    enable_accept_encoding_brotli = true
    enable_accept_encoding_gzip   = true
  }
}
{% endhighlight %}

The above is essentially the same as the built-in `Managed-CachingOptimized`. We can attach this to a cache behaviour block through its `cache_policy_id` argument.

After the distrbution has been deployed, we can retrieve the distribution url and make some requests to retrieve some assets to check the headers.

To test in the terminal using `curl` we need to set the `Accept-Encoding` header value, setting `br,gzip` as its value. 

For example to test access to a javascript via cli:
```
curl -v -I -H "Accept-Encoding: br,gzip,deflate" ${CDN_URL}/jquery.min.js
```

If compression is working, you should see in the response headers `vary; Accept-Encoding` header and a `content-encoding: br` as we specified brotli as the first option. Replacing it with `gzip` should return `content-encoding: gzip`. Subsequent requests should transition from `x-cache: Miss from cloudfront` to `x-cache: Hit from cloudfront`

There are certain conditions in which compression won't work:

* If the asset has already been cached in cloudfront and compression is switched on after, cloudfront will keep returning the cached asset until it expires before returning a fresh copy with compression enabled.

* Only certain resource types can be compressed. Please check the list of [Supported Compression FileTypes].

* If the origin s3 bucket is configured as a website endpoint, we can't use cloudfront as S3 doesn't support HTTPS connections in that configuration

* Cloudfront compression is set to `off` by default. This needs to be set to `on` before compression works.

The complete terraform code example is as follows:
<script src="https://gist.github.com/cheeyeo/e932b195040994c33693ed748be7483f.js"></script>

In this post, I aim to describe how to setup a CDN using Cloudfront distribution with a private s3 bucket as its origin.

There are other areas which are not covered in this post:

* Using a signed URLs or signed cookies to restrict access to cached files. 

* Using a staging distribution to test CDN.

* Multiple cache behaviour based on different policy to improve cache hit ratio

More information can be found at [Private Content Task List] and [Optimizing caching and availability]

Hope it helps someone.

H4ppy H4ck1n6 !!! 