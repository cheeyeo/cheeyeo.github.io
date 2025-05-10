---
layout: post
show_meta: true
title: Using AWS CodePipeline to automate EC2 deployment
header: Using AWS CodePipeline to automate EC2 deployment
date: 2025-05-02 00:00:00
summary: Using AWS CodePipeline to automate EC2 deployment via CodeDeploy Agent
categories: aws codepipeline codebuild codedeploy CI/CD pipeline
author: Chee Yeo
---

In a previous post, I explained how one could utilize the AWS Codepipeline, CodeBuild and CodeDeploy to run a blue/green deployment of a service running on ECS. This article aims to explain a similar process but for EC2 instances.

As a pre-requisite, we need a running EC2 Instance with a tag of `Name: CodeDeployDemo`. The EC2 Instance role needs to have the `SSM` core instance role attached in order to use SSM as well as a policy to read the CodePipeline s3 artifacts directory. 

We need to install the `CodeDeploy Agent` and the recommended approach is to use SSM Distributor to install using `RunCommand` with the `AWS CodeDeploy config`. Once installed, we can use SSM to login to the instance and check its status:
```
sudo systemctl status codedeploy-agent
```

You can also view the logs at ``. Further posts will explore how to stream logs into CloudWatch.

Follwing the agent install, the following resources need to be implemented for it to work:
* CodeDeploy EC2 Service Role
* CodeDeploy application and deployment group
* CodePipeline pipeline

For creating the EC2 Service role for CodeDeploy, we need to attach the `AWSCodeDeployRole` policy with a role that has a trust policy of:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Service": "codedeploy.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```
This role needs to be attached to the EC2 instance.

The deployment requires an `appspec.yml` file which has the following:
```
version: 0.0
os: linux
files:
  - source: WordPress
    destination: /var/www/html/WordPress
  - source: scripts
    destination: scripts
hooks:
  AfterInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root
    - location: scripts/change_permissions.sh
      timeout: 300
      runas: root
  ApplicationStart:
    - location: scripts/start_server.sh
    - location: scripts/create_test_db.sh
      timeout: 300
      runas: root
  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 300
      runas: root
```

The above is an example adapted from CodeDeploy for deploying a LAMP stack running wordpress. This file is in the root of the github repo. The specification above copies the wordpress source and copies it to `/var/www/html` which is the default directory for Apache web server. By default, the working directory for the Codedeploy agent is at `/opt/codedeploy-agent/deployment-root`. The scripts sub-dir will be copied to `/opt/codedeploy-agent/deployment-root/deployment-group-id/deployment-id/deployment-archive`. Using the relative paths, we could run the scripts after they are copied. The `AfterInstall` scripts install the WP deps such as httpd and sets up the database. The `ApplicationStart` hooks starts the httpd server and creates a test database. The `ApplicationStop` hooks run when it receives a stop lifecycle event and this stops the httpd server.

For EC2 and on-prem instances, there are 3 in place deployment strategies:

* CodeDeployDefault.AllAtOnce
  Replaces all the instances immediately with the new revision

* CodeDeployDefault.HalfAtATime
  Replace half of the instances at a time with the new revision.

* CodeDeployDefault.OneAtATime
  Deploys the new revision to one instance at a time.

We use the default of `CodeDeployDefault.OneAtATime`. The deployment configuration also supports blue/green dpeloyment but it requires an autoscaling group and load balancer which is outside the scope of this article.

To create a new CodePipeline pipeline, we only use the following two stages:

* Source
* Deploy

When creating the CodePipeline in console, we can use a new service role but we also need to add in the `GetApplicationRevision` policy else it will fail with permissions error...

The `Source` stage links to the github repo where the scripts and appspec.yml files are stored. The outputs from this stage are uploaded as a zip bundle in an S3 bucket. This bundle is downloaded by the CodeDeploy agent during deployment and extracted on the instance. As per the appspec.yml file `files` block, the Wordpress files are extracted to `/var/www/html` and the scripts are extracted to the agent working directory. After the `Install` lifecycle has completed, the files will be moved and we can run the setup scripts. The deployment event logs will show the progress and status of each of the hooks.

< screenshot of deployment events >




