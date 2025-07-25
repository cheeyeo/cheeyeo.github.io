---
layout: post
show_meta: true
title: Applying Application Autoscaling with Target Tracking policy for ECS service
header: Applying Application Autoscaling with Target Tracking policy for ECS service
date: 2025-07-25 00:00:00
summary: How to apply application autoscaling using Target Tracking policy for a flask service on ECS
categories: flask python ecs autoscaling
author: Chee Yeo
---

[Application Autoscaling]: https://docs.aws.amazon.com/autoscaling/application/userguide/what-is-application-auto-scaling.html

[Target Tracking policy for ECS]: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/target-tracking-create-policy.html

When working with EC2 instances, we can create an autoscaling group to scale in or out a group of instances based on certain conditions such as CPU usage. We can apply the same approach to scaling application services through the use of **Application Autoscaling**

[Application Autoscaling] is an AWS service that automatically scales applications and services based on the metrics you want to track. Once the metrics exceed a specific threshold or a condition is met, this triggers the autoscaling policy which either scales in or scales out. There are four main types of scaling policies:

*  Target tracking policy. This scales a resource based on a target value for a specific Cloudwatch metric such as CPU or memory. It doesn't require creating a Cloudwatch alarm.

* Step tracking policy. Scales a resource based on a set of scaling adjustments that vary based on size of alarm breach. Requires a Cloudwatch Alarm for the metric.

* Scheduled scaling policy. Scales a resource once only or on a recurring schedule.

* Predictive scaling policy. Scales resource based on historical data to match anticipated load.

This post will show an example of applying **Target Tracking** to an ECS service. 

We could apply autoscaling after the service has been created either in the console or via the CLI. We can also create the autoscaling resources via Terraform at the same time as creating the ECS service.

Firstly, we need to create an IAM role that will be assumed by application autoscaling. This role needs to assume **application-autoscaling.amazonaws.com** and have both ECS service and cloudwatch logs permissions:

{% highlight terraform %}
resource "aws_iam_role" "ecs_autoscaling_role" {
  name = "ECSAutoscalingRole"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
        Principal = {
          Service = "application-autoscaling.amazonaws.com"
        }
      },
    ]
  })
}

data "aws_iam_policy_document" "ecs_autoscaling" {
  statement {
    effect = "Allow"
    actions = [
      "ecs:DescribeServices",
      "ecs:UpdateService",
      "cloudwatch:PutMetricAlarm",
      "cloudwatch:DescribeAlarms",
      "cloudwatch:DeleteAlarms"
    ]
    resources = [
      "arn:aws:ecs:eu-west-2:${data.aws_caller_identity.current.account_id}:*/*",
      "arn:aws:cloudwatch:eu-west-2:${data.aws_caller_identity.current.account_id}:*/*"
    ]
  }
}

resource "aws_iam_role_policy" "ecs_autoscaling" {
  name   = "AllowECSAutoscaling"
  role   = aws_iam_role.ecs_autoscaling_role.name
  policy = data.aws_iam_policy_document.ecs_autoscaling.json
}
{% endhighlight %}

Next, we need to create an **application autoscaling target** which will be the ECS service. We define the desired count from the service as the scalable attribute and specifies a min and max capacity values. This is similar to how Autoscaling groups are configured for EC2 Instances:

{% highlight terraform %}
resource "aws_appautoscaling_target" "ecs_target" {
  max_capacity       = 4
  min_capacity       = 1
  resource_id        = "service/${aws_ecs_cluster.flaskapp.name}/${aws_ecs_service.flaskapp.name}"
  role_arn           = aws_iam_role.ecs_autoscaling_role.arn
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}
{% endhighlight %}

Lastly, we create and apply an **application autoscaling policy** to the **autoscaling target** defined above. Each type of autoscaling policy has its own policy configuration block. For **targeted tracking scaling**, we need to define the metric type to measure. According to [Target Tracking policy for ECS], we have 3 types of metrics available: **ECSServiceAverageCPUUtilization, ECSServiceAverageMemoryUtilization, ALBRequestCountPerTarget**. The example below shows the policy for scaling based on CPU usage:

{% highlight terraform %}
resource "aws_appautoscaling_policy" "ecs_policy" {
  name               = "cpu-scale-out"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }

    target_value       = 10.0
    scale_in_cooldown  = 60
    scale_out_cooldown = 60
  }
}
{% endhighlight %}

It is also recommended to create a separate policy for memory usage:
{% highlight terraform %}
resource "aws_appautoscaling_policy" "ecs_mem_policy" {
  name               = "memory-scale-out"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageMemoryUtilization"
    }

    target_value       = 10.0
    scale_in_cooldown  = 60
    scale_out_cooldown = 60
  }
}
{% endhighlight %}

The **target_value** specifies the limit upon which the scaling starts. It's set to a test value of 10.0 for testing purposes. The scale in and scale out cooldown values refer to the amount of time to wait before the next scaling event can occur.

Once applied, the scaling configuration is listed under the `ECS Cluster > Service > Service autoscaling` tab as shown below:

![Scaling policy tab](/assets/img/aws/application_autoscaling/scaling_policy.png)

To test the scaling policy, I logged into the application and triggered the memory scale out alarm which resulted in an additional task being provisioned, increasing the desired count to 2:

![Memory scale out](/assets/img/aws/application_autoscaling/memory_scale_out.png)
![Memory scale out](/assets/img/aws/application_autoscaling/memory_scale_out_activity.png)


The **Events** tab of the service should show the scaling activity occuring. We can see that the number of tasks have increased by one:

![Scaling Activities](/assets/img/aws/application_autoscaling/scaling_activities.png)
