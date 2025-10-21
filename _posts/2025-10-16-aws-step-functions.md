---
layout: post
show_meta: true
title: AWS Step Functions
header: AWS Step Functions
date: 2025-10-16 00:00:00
summary: Short intro to AWS Step Functions
categories: python aws step-functions serverless lambda
author: Chee Yeo
---

[AWS Step Functions]: https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html
[Article from datacamp]: https://www.datacamp.com/tutorial/mastering-aws-step-functions-a-comprehensive-guide
[JSONata website]: https://docs.jsonata.org/overview.html

[AWS Step Functions] allows you to create workflows called **State Machines** which allows you to coordinate and orchestrate AWS services to perform tasks and automate processes. A State Machine consists of stages linked together in a directed acyclic graph (DAG) which means the flow of information is uni-directional in one direction from one stage to the next. 

Each step in a state machine is called a **state**. There are 2 types of states:
* Flow states
* Task states

Flow states control the flow of execution. For example, we can have a step which has a **Choice** state that can invoke one of 2 subsequent steps based on its input from the previous step. 

Task states perform a unit of work. This is delegated to an AWS service such as Lambda or an external service. 

Data is passed into the workflow in JSON format. Each step passes data to the next using state output and variables. Variable data can be stored and be reused as input in later steps. 

State Machines can be defined using either JSONPath or JSONata. Only JSONata supports the newer features such as variables, arguments and assignment. For this article, we will be using the newer JSONata syntax. More in depth study of JSONata is out of the scope of this article.

The screenshot below shows the state machine based on an [Article from datacamp]. The problem domain involves passing daycare registration information to the state machine which performs a series of data validation and if successful, proceeds to the registration stage. I refactored the state machine definition to use the newer JSONata syntax.

![State Machine](/assets/img/aws/stepfunctions/stepfunctions_graph.png)

The Task states are implemented using external lambda functions. The Flow states determine whether the registration process should continue based on the outputs of previous states.

The state machine starts at the **Check Information** stage which checks that the input data has certain keys in the body. The definition of this stage is as follows:

{% highlight json %}
{% raw %}
"Check Information": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Output": "{% $states.result.Payload %}",
      "Arguments": {
        "FunctionName": "arn:aws:lambda:eu-west-2:035663780217:function:CheckInformation:$LATEST",
        "Payload": "{% $states.input %}"
      },
      "Assign": {
        "statusCode": "{% $states.result.Payload.statusCode %}"
      },
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException",
            "Lambda.SdkClientException",
            "Lambda.TooManyRequestsException"
          ],
          "IntervalSeconds": 1,
          "MaxAttempts": 3,
          "BackoffRate": 2,
          "JitterStrategy": "FULL"
        }
      ],
      "Next": "Information Check Result"
    }
{% endraw %}
{% endhighlight %}

The task state specifies the `CheckInformation` lambda as an entrypoint. It assigns the input to the state machine as the lambda payload. The output of the lambda is set in the `states.result.Payload` body. We use the returned status code to assign to a variable **statusCode** which we use in the next stage **Information Check Result**. The lambda code is as follows:

{% highlight python %}
import json
import datetime


def lambda_handler(event, context):
    registration_info = event['registration_info']

    required_fields = ['child', 'parents', 'daysOfWeek']
    for field in required_fields:
        if field not in registration_info:
            return {
                'statusCode': 400,
                'body': f"Missing required field: {field}"
            }
    
    return {
        "statusCode": 200,
        "body": json.dumps(registration_info)
    }
{% endhighlight %}

Note how the returned body is mapped to the `states.result.Payload`. To access `statusCode` we use `states.result.Payload.statusCode`

The `states.input` is only available in the entry stage but can still be accessed in subsequent stages using `states.context.Execution.Input`

The next stage **Information Check Result** is a flow state that uses a `Choice` type to determine which stage to select next. If the status code is 200, it calls the next stage **Check age range** else it calls **Notify Missing Info** which is a **Fail** flow type. The defintion of the stage is as follows:

{% highlight json %}
{% raw %}
    "Information Check Result": {
      "Type": "Choice",
      "Choices": [
        {
          "Condition": "{% $statusCode = 200 %}",
          "Next": "Check age range"
        },
        {
          "Condition": "{% $statusCode = 400 %}",
          "Next": "Notify Missing Info"
        }
      ]
    }
{% endraw %}
{% endhighlight %}

If the value of `statusCode` variable is 200, it calls `Check age range` stage, which is a Task state with a lambda function. If the `statusCode` variable is not 200, it calls `Notify Missing Info` which is a Flow stage of type **Fail** as shown below:

{% highlight json %}
    "Notify Missing Info": {
      "Type": "Fail",
      "Error": "InformationIncomplete",
      "Cause": "The parent did not provide complete information."
    }
{% endhighlight %}

The subsequent stages are of a similar structure with a lambda Task stage followed by a Flow stage. The final stage is a Flow stage of type **Succeed** which just outputs a message if registration is successful.

The full state machine defintion and lambda code is provided in this gist:
<script src="https://gist.github.com/cheeyeo/c01f69c64488d9662af44f85512cb916.js"></script>


To test the state machine, we create an `Execution` and pass in the sample data as input. The screenshots below shows a success invocation and a failure when the sample data fails validation.

![State Machine Execution](/assets/img/aws/stepfunctions/execution.png)
![State Machine Execution Success](/assets/img/aws/stepfunctions/execution_success.png)
![State Machine Execution Failure](/assets/img/aws/stepfunctions/execution_failure.png)

Note that we declare the `QueryLanguage` type in the body of the state machine definition to be `JSONata`. This would apply to all the stages. One can also use a hybrid of `JSONPath` and `JSONata` by declaring the `QueryLanguage` keyword in each stage.

The state machine is built in the AWS UI Console using the builder. The definition file is exported from the console and refined before importing back into the console again. Note that the console doesn't save the state machine automatically. You need to hit `create` button manually. During creation, a separate popup will appear asking if it should create a custom IAM role and if it should produce logging to Cloudwatch Logs. To enable the state machine to invoke lambda functions or any services, you need to create a custom IAM role with the required permissions policy. Given the example above, the console created a custom IAM role that grants permissions to invoke the lambda functions and write to Cloudwatch logs. The example below shows the permission policies attached to the IAM role:

{% highlight json %}
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogDelivery",
                "logs:GetLogDelivery",
                "logs:UpdateLogDelivery",
                "logs:DeleteLogDelivery",
                "logs:ListLogDeliveries",
                "logs:PutResourcePolicy",
                "logs:DescribeResourcePolicies",
                "logs:DescribeLogGroups"
            ],
            "Resource": "*"
        }
    ]
}


{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "lambda:InvokeFunction"
            ],
            "Resource": [
                "arn:aws:lambda:eu-west-2:XXXX:function:CheckInformation:*",
                "arn:aws:lambda:eu-west-2:XXXX:function:CheckAgeRange:*",
                "arn:aws:lambda:eu-west-2:XXXX:function:CheckSpotAvailable:*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "lambda:InvokeFunction"
            ],
            "Resource": [
                "arn:aws:lambda:eu-west-2:XXXX:function:CheckInformation",
                "arn:aws:lambda:eu-west-2:XXXX:function:CheckAgeRange",
                "arn:aws:lambda:eu-west-2:XXXX:function:CheckSpotAvailable"
            ]
        }
    ]
}

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "xray:PutTraceSegments",
                "xray:PutTelemetryRecords",
                "xray:GetSamplingRules",
                "xray:GetSamplingTargets"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
{% endhighlight %}

The console added additional permissions policy for the X-Ray service.

Step Functions is a powerful and complex service to utilize. I still struggle with the state machine definition. With JSONata, it should help reduce the complexity of the developing the workflow. This will be explored in future posts, where we will attempt to remove some of the lambda task states and refactor them to use JSONata syntax only.

More information on JSONata can be found on the [JSONata website]
