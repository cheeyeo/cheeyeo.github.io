---
layout: post
show_meta: true
title: Using Autoscaling Group waiter in AWS SDK v2
header: Using Autoscaling Group waiter in AWS SDK v2
date: 2024-10-01 00:00:00
summary: Using Autoscaling Group waiter in AWS SDK v2
categories: go-lang aws sdk
author: Chee Yeo
---

In a recent infrastructure project, I had to provision an autoscaling group using Terraform. However, I couldn't work out how to wait until the ASG is ready with its instances attached in terraform. After some research, it seems that the recommended approach is to leverage lifecycle hooks and let AWS handle this through events. While this is a scalabale approach, I feel that its too complicated for my use case which was to just wait till the ASG is in a ready state with instances attached. I don't need to perform any action on those instances.

I decided to create an external script in go in order to make use of the AWS Go-Lang SDK v2. The provided module `github.com/aws/aws-sdk-go-v2/service/autoscaling` allows you to create an `InServiceWaiter` which will return once the autoscaling group has reached a Ready state with the instances attached.

My initial attempt was as follows:

{% highlight golang linenos %}
package main

import (
    "context"
	"fmt"
	"log"
	"os"
	"sync"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/autoscaling"
)


func main() {
    cfg, err := config.LoadDefaultConfig(context.TODO())
	if err != nil {
		log.Fatal(err)
	}

	client1 := autoscaling.NewFromConfig(cfg)
	param := &autoscaling.DescribeAutoScalingGroupsInput{
		AutoScalingGroupNames: []string{
			os.Getenv("ASG"),
		},
	}

    waiter := autoscaling.NewGroupInServiceWaiter(client1)
	waiter.Wait(context.TODO(), param, time.Duration(300*float64(time.Second)))

    fmt.Println("ASG is ready")

    ...
}
{% endhighlight %}

In the code above, we initialized a client for autoscaling and create a new waiter via `NewGroupInServiceWaiter`. It takes in a context struct, a parameter with the name of the autoscaling group to wait for and a maximum time out value of 300 seconds or 5 minutes.

The above is then wrapped in a `terraform_data` resource and invoked via a `local-exec` provisioner:

{% highlight terraform linenos %}
resource "aws_autoscaling_group" "group" {
  default_cooldown     = 300
  health_check_type    = "ELB"
  termination_policies = ["OldestInstance"]
  desired_capacity     = 3
  max_size             = 3
  min_size             = 1

  launch_template {
    id      = aws_launch_template.vault_template.id
    version = "$Latest"
  }

  name = "vault-dev"

  tag {
    key                 = "Name"
    propagate_at_launch = true
    value               = local.instance_name
  }

  target_group_arns   = [aws_lb_target_group.tstvault[0].arn]
  vpc_zone_identifier = module.vpc.private_subnets

  instance_refresh {
    preferences {
      instance_warmup        = 300
      min_healthy_percentage = 90
    }

    strategy = "Rolling"
  }

  lifecycle {
    ignore_changes = [desired_capacity, target_group_arns]
  }

  timeouts {
    delete = "15m"
  }

  wait_for_capacity_timeout = "0"
}

resource "terraform_data" "custom" {
  depends_on = [ aws_autoscaling_group.group ]

  provisioner "local-exec" {
    command = "cd ${path.cwd}/scripts && ASG=\"${aws_autoscaling_group.group.name}\" go run asg.go"
  }
}
{% endhighlight %}

However, the above skips the waiter process. As soon as the autoscaling group resource is created, it calls `terraform_data.custom` and invokes the `asg.go` script, which invokes the waiter but immediately calls the code after it, causing the deployment to fail.

After some investigation, it turns out that the default waiter in the autoscaling module in the SDK doesn't wait for the instances to be ready but returns as soon as the autoscaling group is created.

We can create a custom retryable function which we can pass into the initialization of the waiter to make it wait until the required conditions are met:

{% highlight golang linenos %}
customRetryable := func(ctx context.Context, params *autoscaling.DescribeAutoScalingGroupsInput, output *autoscaling.DescribeAutoScalingGroupsOutput, err error) (bool, error) {
		if len(output.AutoScalingGroups[0].Instances) < 1 {
			return true, nil
		}

		return false, nil
	}
{% endhighlight %}

The retry function needs to fit the signature of `func(ctx context.Context, params *autoscaling.DescribeAutoScalingGroupsInput, output *autoscaling.DescribeAutoScalingGroupsOutput, err error) (bool, error)` in order to pass it as additional options to the waiter. In my use case, I am only interested in the number of instances attached. If it's less than 1, we continue to wait. If not, we can exit the waiter and continue.

To pass the retryable function into the waiter:

{% highlight golang linenos %}
waiter := autoscaling.NewGroupInServiceWaiter(client1, func(o *autoscaling.GroupInServiceWaiterOptions) {
		o.Retryable = customRetryable
	})
{% endhighlight %}

The complete code listing:

{% highlight golang linenos %}
package main

import (
    "context"
	"fmt"
	"log"
	"os"
	"sync"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/autoscaling"
)


func main() {
    cfg, err := config.LoadDefaultConfig(context.TODO())
	if err != nil {
		log.Fatal(err)
	}

	client1 := autoscaling.NewFromConfig(cfg)
	param := &autoscaling.DescribeAutoScalingGroupsInput{
		AutoScalingGroupNames: []string{
			os.Getenv("ASG"),
		},
	}

    customRetryable := func(ctx context.Context, params *autoscaling.DescribeAutoScalingGroupsInput, output *autoscaling.DescribeAutoScalingGroupsOutput, err error) (bool, error) {
		if len(output.AutoScalingGroups[0].Instances) < 1 {
			return true, nil
		}

		return false, nil
	}

    waiter := autoscaling.NewGroupInServiceWaiter(client1, func(o *autoscaling.GroupInServiceWaiterOptions) {
		o.Retryable = customRetryable
	})

	waiter.Wait(context.TODO(), param, time.Duration(300*float64(time.Second)))

    fmt.Println("ASG is ready")

    ...
}
{% endhighlight %}

With the above, the go lang script runs properly and will invoke the waiter until the instances are in a ready state.

Note that we can apply custom retryable logic to most interfaces provided by the AWS. More information can be found on the documentation:

[AWS SDK V2 USING WAITERS]

[AWS SDK V2 USING WAITERS]: https://aws.github.io/aws-sdk-go-v2/docs/making-requests/#using-waiters

