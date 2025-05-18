---
layout: post
show_meta: true
title: Forward CodeDeploy Agent logs to Cloudwatch Logs
header: Forward CodeDeploy Agent logs to Cloudwatch Logs
date: 2025-05-15 00:00:00
summary: Forward CodeDeploy Agent logs to Cloudwatch Logs
categories: aws codepipeline codebuild codedeploy CI/CD pipeline cloudwatch ec2
author: Chee Yeo
---

[CodeDeploy agent logs to CloudWatch Logs]: https://docs.aws.amazon.com/codedeploy/latest/userguide/codedeploy-agent-operations-cloudwatch-agent.html

[Install CodeDeploy agent via SSM]: https://docs.aws.amazon.com/codedeploy/latest/userguide/codedeploy-agent-operations-install-ssm.html

[Install Cloudwatch agent via SSM]: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/download-CloudWatch-Agent-on-EC2-Instance-SSM-first.html

[Cloudwatch Agent Configuration File]: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html


While working with CodeDeploy on EC2, one of the issues was examining the logs to debug a failed deployment. By default, the codedeploy agent logs to `/var/log/aws/codedeploy-agent/codedeploy-agent.log` and one could login via SSM to the EC2 instance to examine the log file. A better approach would be to stream the [CodeDeploy agent logs to CloudWatch Logs] via the CloudWatch agent. 

Note that there are different approaches on installing the CodeDeploy and Cloudwatch agent from the docs. To simplify the process, I'm documenting the same steps here I took for installing the Cloudwatch agent, which is via this documentation: [Install Cloudwatch agent via SSM]

The first step is to install the CodeDeploy agent. This could be achieved via `SSM > RunCommand` and selecting the `AWS-ConfigureAWSPackage` command document. Under the name field we specify the name of the software package which is `AWSCodeDeployAgent`. Specify the name of the cloudwatch log group to monitor the progress of the install. Once completed, verify the agent is running from the EC2 instance

![CodeDeploy](/assets/img/codepipeline-ec2-logs/img1.png)
![CodeDeploy](/assets/img/codepipeline-ec2-logs/img2.png)

The second step is to install the CloudWatch agent. We are going to use the same process as above except we are replacing the name field with `AmazonCloudWatchAgent`

![CodeDeploy](/assets/img/codepipeline-ec2-logs/img3.png)

By default, the cloudwatch agent will not start unless a valid configuration file is provided. Checking on the status of the cloudwatch agent will show the following:

{% highlight shell %}
sh-5.2$ sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
{
  "status": "stopped",
  "starttime": "",
  "configstatus": "not configured",
  "version": "1.300055.0b1095"
}
{% endhighlight %}


[Cloudwatch Agent Configuration File] specifies the format for the config file. This is in the form of a JSON document and consists of four main sections:

* agent
  > For overall configuration of the agent

* metrics
  > Specify custom metrics for collection and publish to Cloudwatch logs. This can be omitted if only using the agent to collect logs.

* logs
  > Specifies which logs files are published to cloudwatch logs.

* traces
  > Specifies sources for traces that are sent to AWS X-Ray


The example config file we used below specifies the CodeDeploy agent logs, its location on disk, and the log group and stream to send to:

{% highlight json %}
{
    "logs": {
        "logs_collected": {
            "files": {
                "collect_list": [
                    {
                        "file_path": "/var/log/aws/codedeploy-agent/codedeploy-agent.log",
                        "log_group_name": "testpipeline",
                        "log_stream_name": "{instance_id}-agent-log"
                    },
                    {
                        "file_path": "/opt/codedeploy-agent/deployment-root/deployment-logs/codedeploy-agent-deployments.log",
                        "log_group_name": "testpipeline",
                        "log_stream_name": "{instance_id}-codedeploy-agent-deployment-log"
                    },
                    {
                        "file_path": "/tmp/codedeploy-agent.update.log",
                        "log_group_name": "testpipeline",
                        "log_stream_name": "{instance_id}-codedeploy-agent-updater-log"
                    }
                ]
            }
        }
    }
}
{% endhighlight %}

The `{instance-id}` is a placeholder which will be replaced with the actual instance ID on startup. The configuration is forwarding the codedeploy agent logs to a cloudwatch log group name of `testpipeline` into its individual log streams, depending on the type of logs. For example, any update to the agent goes to the `{instance_id}-codedeploy-agent-updater-log` whereas any deployments go to `{instance_id}-codedeploy-agent-deployment-log`.

To apply the above config to the Cloudwatch agent, we need to store is as a `STRING` type in SSM Parameter Store. This can be achieved via this CLI command:

{% highlight shell %}
aws ssm put-parameter --name "testagentcfg" --type "String" --value file://cloudwatch.json
{% endhighlight %}

Assuming that we saved the above config file as `cloudwatch.json` we can use the `put-parameter` CLI command to save it.

The next step is to run the `AmazonCloudWatch-ManageAgent` Run Command, which will take as input the SSM parameter value for the configuration file and configure the agent. Under `Optional Configuration Store` we select `ssm` and provie the name of the parameter under `Optional Configuration Location`:

![CodeDeploy](/assets/img/codepipeline-ec2-logs/img4.png)

If applied successfully, the cloudwatch agent should be active:
{% highlight shell %}
sh-5.2$ sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
{
  "status": "running",
  "starttime": "2025-05-18T14:57:35+00:00",
  "configstatus": "configured",
  "version": "1.300055.0b1095"
}
{% endhighlight %}

We should also see the log streams collated under the log group:

![CodeDeploy](/assets/img/codepipeline-ec2-logs/img7.png)

![CodeDeploy](/assets/img/codepipeline-ec2-logs/img5.png)

Any deployments via CodeDeploy and CodePipeline can be viewed in the log stream `{instance_id}-codedeploy-agent-deployment-log`:

![CodeDeploy](/assets/img/codepipeline-ec2-logs/img6.png)
