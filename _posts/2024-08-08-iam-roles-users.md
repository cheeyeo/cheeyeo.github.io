---
layout: post
show_meta: true
title: IAM Roles vs IAM users in AWS
header: IAM Roles vs IAM users in AWS
date: 2024-08-08 00:00:00
summary: Difference between IAM roles and users in AWS
categories: aws iam
author: Chee Yeo
---

AWS IAM consists of the following core components:

* IAM User
* IAM Groups
* IAM Roles
* Policies

An IAM user is an actual user created within the AWS account. It has its own set of credentials ( access key, secret access key ) which can be used to access the AWS API. This is what most users refer to when discussing IAM. This is also the main method of accessing AWS services as an end-user. When we run `aws configure`, we require the credentials from the IAM user in order to authenticate to the platform.

An IAM group is a collection of IAM users who share similar responsbilities and access. For example, we could create an IAM group called `devops` who have admin privileges to manage infrastructure on the platform. Any roles assigned to an IAM group propagates down to all users in the group.

An IAM role is used for authorization. It's associated with AWS resources but can also be used by users. It's a container that consists of IAM permission policies and a trust policy. The IAM permission policies grant permission to the role to perform certain actions on resources such as listing S3 buckets. The trust policy determines who can assume this role. When the role is attached to a resource, it calls the Simple Token Service (STS) to obtain short-lived credentials to make AWS API calls to perform certain actions.

For example, given a basic Lambda execution role, it would have a trust policy to allow the lambda resource to obtain credentials from STS such as:

{% highlight json %}
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": "lambda.amazonaws.com"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }
{% endhighlight %}

This is an important point to remember: **all actions on the AWS platform rely on AWS credentials such as secret access token and access token**. Even when we sign in to the platform via external providers such as Active Directory, the directory connector to AWS would need to exchange user credentials with actual AWS credentials before the user can make any AWS API calls.

Policies define the permissions that a role is allowed to perform. These policies are attached to the role in the form of JSON policy documents. Each policy specify the actions allowed to perform on a resource. Policies can be customer / user defined or provided by AWS. A role can have multiple policies attached to it.

For example, we would attach an AWS policy of `AWSLambdaBasicExecution` policy to a Lambda role as it contains the minimal permissions allowed for any Lambda function, which is the ability to create a log group and write to a log stream within it for basic logging:

{% highlight json %}
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
{% endhighlight %}

Roles can also be used by IAM users to access other accounts. This is known as cross-account roles. For example, when we create an AWS organization, any user account created directly within an OU ( Organization Unit ) has a `OrganizationAccountAccessRole` created by default. This allows the administrator of the organization to `Switch Role` to administer the account. The trust policy of such a role is shown below:

{% highlight json %}
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::XXXXX:root"
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}
{% endhighlight %}

Notice that the principal refers to an AWS account ARN rather than to a service. The root suffix means any user with admin rights from account XXXXX is allowed to assume this role via Switch Role. Like other roles, additional permissions can also be attached to cross-account roles.

A difference between IAM users and roles is in the validity period of the credentials. IAM user's credentials can be created/revoked manually via the AWS console. IAM role's credentials are short-lived ( valid for 36 hours ) and provided by the STS service.

Another difference is the scope of how it's used. IAM users are used to authenticate to the platform whereas IAM roles are used for authorization.