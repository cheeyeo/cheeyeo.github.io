---
layout: post
show_meta: true
title: Using go-routines with waiters in AWS SDK V2
header: Using go-routines with waiters in AWS SDK V2
date: 2024-10-08 00:00:00
summary: Using go-routines with waiters in AWS SDK V2
categories: go-lang aws sdk waiters go-routines
author: Chee Yeo
---

In a recent project which uses the AWS SDK V2 for go-lang, I had to devise a strategy to wait for a single EC2 instance within an autoscaling group to be in a ready state. A separate action will invoke SSM RunCommand to execute on this instance to perform some setup. 

After researching on suitable patterns to use, I realised that most of the examples in the wild point to applying the same set of scripts or actions on all the instances in a group. As such, I had to devise a custom script to perform the above.

Since I need to check on all the instances simultaneously, I need to run each check in a separate go-routine and exit as soon as the instance reaches a ready state. I also need to stop the other waiters on the remaining instances. 

My initial attempt is as follows:

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
	"github.com/aws/aws-sdk-go-v2/service/ec2"
)

func main() {
    cfg, err := config.LoadDefaultConfig(context.TODO())
    if err != nil {
        log.Fatal(err)
    }

    // additional code to wait for the autoscaling group...


    var instances []string
    for _, r := range resp.AutoScalingGroups[0].Instances {
        instances = append(instances, *r.InstanceId)
    }


    ctx, cancelFunc := context.WithCancel(context.Background())
	defer cancelFunc()

    // stores the ready instance name
    ch := make(chan string)

    ec2_client := ec2.NewFromConfig(cfg)

    var wg sync.WaitGroup
	wg.Add(len(instances))

    for _, inner_instance := range instances {
		go func(instance string) {
			defer wg.Done()

			waiter := ec2.NewInstanceStatusOkWaiter(ec2_client)
			param := &ec2.DescribeInstanceStatusInput{
				InstanceIds: []string{
					instance,
				},
			}

			err := waiter.Wait(context.TODO(), param, time.Duration(300*float64(time.Second)))

			if err != nil {
				fmt.Println("Error In go-routine ", err)
				cancelFunc()
				return
			}

			select {
			case ch <- instance:
				// Exits waiter loop for first instance which is ready
				fmt.Println("In go routine: ", instance)
				cancelFunc()
			case <-ctx.Done():
			}

		}(inner_instance)
	}

    var mainInstance string

loop:
  	for {
		select {
		case s := <-ch:
			mainInstance = s
			fmt.Println("In main: ", mainInstance)
		case <-ctx.Done():
			fmt.Println("Cancelled")
			break loop
		}
	}
	wg.Wait()

    fmt.Println("Running setup on ", mainInstance)
}
{% endhighlight %}

The pattern we want to use here is to create a group of go-routines to run the status checks and exit each one of them as soon as any one of the instances has reached a ready state.

We create a `sync.WaitGroup` to synchronize the work across all the go-routines. Within a for loop, a separate go-routine is created for each EC2 instance. Within the go-routine, we create a `NewInstanceStatusOkWaiter` which takes as parameters a context struct; the instance ID to wait; and a max duration of 300 seconds for timeouts. We check the returned error from creating a waiter and if it exists, we immediately send a signal to stop the go-routine and exit. The `select` clause within each go-routine controls its behaviour. When an instance is ready, it will send its instance ID to the `ch` channel in the first case statement. This calls `cancelFunc()` which places a signal on `ctx.Done` causing it to exit. Likewise, all the remaining go-routines will also be waiting for a signal on the `ctx.Done` channel and when notified, it will also exit the go-routine and fall into the second statement of the outer loop.

When an instance reaches a ready state, it will send its instance ID to the `ch` channel which calls `cancelFunc`. Note that we created a `context.WithCancel` which returns a ctx struct and a cancel function. When we call the cancel function, it sends a signal to `ctx.Done` which causes the go-routines to stop and break out of the `loop` block in the main thread.

While the above code works in getting the instance ID of the first ready instance, it will still wait on the remaining waiters to complete or time out before it finally exits the wait loop. This is because when we created the go-routines, we are using a default context of `context.TODO`. If we passed in the context created by `context.WithCancel` instead, when the first instance sends a cancel signal, it will signal `ctx.Done` and the remaining waiters will stop immediately and call the second case statement in outer `loop`, breaking out of the loop. The instance ID gets stored in `mainInstance` for further processing.

The modified code now becomes:

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
	"github.com/aws/aws-sdk-go-v2/service/ec2"
)

func main() {
    cfg, err := config.LoadDefaultConfig(context.TODO())
    if err != nil {
        log.Fatal(err)
    }

    // additional code to wait for the autoscaling group...


    var instances []string
    for _, r := range resp.AutoScalingGroups[0].Instances {
        instances = append(instances, *r.InstanceId)
    }


    ctx, cancelFunc := context.WithCancel(context.Background())
	defer cancelFunc()

    // stores the ready instance name
    ch := make(chan string)

    ec2_client := ec2.NewFromConfig(cfg)

    var wg sync.WaitGroup
	wg.Add(len(instances))

    for _, inner_instance := range instances {
		go func(instance string) {
			defer wg.Done()

			waiter := ec2.NewInstanceStatusOkWaiter(ec2_client)
			param := &ec2.DescribeInstanceStatusInput{
				InstanceIds: []string{
					instance,
				},
			}

			err := waiter.Wait(ctx, param, time.Duration(300*float64(time.Second)))

			if err != nil {
				fmt.Println("Error In go-routine ", err)
				cancelFunc()
				return
			}

			select {
			case ch <- instance:
				// Exits waiter loop for first instance which is ready
				fmt.Println("In go routine: ", instance)
				cancelFunc()
			case <-ctx.Done():
			}

		}(inner_instance)
	}

    var mainInstance string

loop:
  	for {
		select {
		case s := <-ch:
			mainInstance = s
			fmt.Println("In main: ", mainInstance)
		case <-ctx.Done():
			fmt.Println("Cancelled")
			break loop
		}
	}
	wg.Wait()

    fmt.Println("Running setup on ", mainInstance)
}
{% endhighlight %}

The change is very subtle but has greater implication. It means that if we ever have to use a waiter in a go-routine, we can use a cancellable context and stop the waiter via the cancel function as shown above.