---
layout:     post
show_meta: true
title:      Using IAM Role with EC2 to access security credentials
header:     Using IAM Role with EC2 to access security credentials
date:       2020-03-11 00:00:00
summary:  How to extract security credentials from IAM Role
categories: aws ec2 iam devops
author: Chee Yeo
---

When we deploy an application onto EC2, it usually needs to access or make API calls to other AWS Services. A common scenario would be an application making calls to S3 service for file uploads. In such cases, we tend to encode the secret access key and access key id as part of the application for the deployment process.

There is another, more secure way to do so in AWS, which is through the use of IAM Roles and EC2 instance metadata.

When we provision an EC2 instance, we have to assign an IAM role to it in order to access other AWS services. However, the security credentials for the role can also be accessed through the Instance Metadata Service within each instance.

This endpoint is of the form `http://169.254.269.254` and involves a curl request. We can obtain both user instance data, dynamic data, and instance specific data by using the following endpoints:

{% highlight console linenos %}
# Access user data passed in during the ec2 provision process
curl http://169.254.169.254/latest/user-data 

# Access instance data
curl http://169.254.169.254/latest/metadata

# Dynamic data
curl http://169.254.169.254/latest/dynamic
{% endhighlight %}

These curl requests are instance specific and can only be run from within the instance itself.

There are two versions of the API, with version 2 being session oriented. The specifics on how they differ and works is outside the scope of this article. More information can be obtained from the [official documentation]{:target="_blank"}.

To access the credentials dynamically, we can do the following:

* Create an IAM Role for the EC2 instance and specify which services you wish to communicate with.

* During deployment, attach the IAM role to the instance. 

* To obtain the security credentials, we can have a user provided script, which is run just after the instance is deployed. We can retrieve and setup the environment variables like so:

{% highlight bash linenos %}
#!/bin/sh

# obtain the IAM role name
ROLE=$(curl http://169.254.169.254/latest/meta-data/iam/security-credentials)

KEYURL="http://169.254.169.254/latest/meta-data/iam/security-credentials/"$ROLE""

# Save the response into a json file
wget $KEYURL -O iam.json

# Use jq CLI parser to retrive the credentials: access key id; secret access key; token

ACCESSKEYID=$(cat iam.json | jq -r '.AccessKeyId')
SECRETACCESSKEY=$(cat iam.json | jq -r '.SecretAccessKey')
TOKEN=$(cat iam.json | jq -r '.Token')

echo $ACCESSKEYID
echo $SECRETACCESSKEY
echo $TOKEN
{% endhighlight %}

The above can be provided as user data during instance provisioning to setup the environment correctly before the application starts.

The credentials are also automatically rotated by AWS, which adds an extra layer of security. This is handled automatically for you when using the AWS CLI or SDK but may have to be handled manually in your custom applications by running a cron job for example.

The downside is the meta-data endpoint is open to any user on the instance and may have to be locked down to only allow access by the root user.

[official documentation]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html