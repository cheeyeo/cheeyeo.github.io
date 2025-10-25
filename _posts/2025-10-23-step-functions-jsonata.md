---
layout: post
show_meta: true
title: Using JSONata functions to simplify Step Functions
header: Using JSONata functions to simplify Step Functions
date: 2025-10-23 00:00:00
summary: How to use JSONata built-in syntax and functions to simply Step Functions
categories: python aws step-functions serverless lambda jsonata
author: Chee Yeo
---

[Simplifying AWS Step functions with JSONata]: https://aws.amazon.com/blogs/compute/simplifying-developer-experience-with-variables-and-jsonata-in-aws-step-functions/
[https://try.jsonata.org]: https://try.jsonata.org/

In my previous post, I provided an example of how to create a step function using Lambdas. The lambda functions serve as data validators to check for required fields in the input data. This created a sequential workflow where the data flows through each lambda and executes the next stage if the data is valid; if not it triggers a stage of type `Fail` which reports an error message.

While this approach works, it can become unwiedly quickly as more data validators are required or the number of pre-processing stages increase. There is a coupling of each lambda validator stage with a corresponding choice stage.

To improve the workflow, I drew inspiration from an AWS blog post on [Simplifying AWS Step functions with JSONata]. I adapted the previous state machine to utilise some of the techniques outlined in the post.

Below is a graph diagram of the revamped state machine from the previous post:

![State Machine](/assets/img/aws/stepfunctions/stepfunctions_graph_v2.png)

Firstly, we moved the data validation stages into the start of the workflow as a type of `Parallel` task. This means we can perform the data validation in parallel without the previous sequential workflow pattern. We also refactored the previous Lambda functions that perform validations to using only inline validations using JSONata. The JSONdata specification provides built-in functions such as count and equality operators we can use to parse and validate the input data.

For example, we replaced the lambda function that check for the presence of key values in the input data using JSONata operators inline:

{% highlight json %}
{% raw %}
{
    "StartAt": "Check Identity",
    "States": {
    "Check Identity": {
        "Type": "Pass",
        "End": true,
        "Output": {
            "isInformationValid": "{% ($count($states.input.registration_info.daysOfWeek) > 0 and $not($states.input.registration_info.daysOfWeek = null)) and $not($states.input.registration_info.child = null) and $not($states.input.registration_info.parents = null) %}"
        }
    }
  }
},
{% endraw %}
{% endhighlight %}

The input json must contain a `registration_info` object with the following attributes: *daysOfWeek; child; parents*. The JSONata function $count checks that the days of week list is not empty. It also checks for the presence of those three attributes which is the same behaviour as the original Lambda function.

Next, we replace the age check Lambda function to be inline:
{% highlight json %}
{% raw %}
{
    "StartAt": "Check Age Range",
    "States": {
    "Check Age Range": {
        "Type": "Pass",
        "End": true,
        "Output": {
            "Age": "{% $number($now('[Y0001]')) - $number($fromMillis($toMillis($states.input.registration_info.child.dateOfBirth), '[Y0001]')) %}"
            }
        }
    }
}
{% endraw %}
{% endhighlight %}

The above uses the built-in `$now()` function to obtain the current datetime and converting it into a year value format and applying `$number()` to convert it into an integer. For converting the input data date of birth, we first convert it into milliseconds using `$toMillis()` and then converting it back to a year format using `$fromMillis()` and specifying the year format. We calculate the difference between the two year values to calculate the age. This is similar in functionality to the Lambda function that checks the age.

Next, we assign the stage outputs to JSONata variables to pass it to later stages:

{% highlight json %}
{% raw %}
"Assign": {
    "inputPayload": "{% $states.context.Execution.Input %}",
    "Age": "{% $states.result.Age %}",
    "isPayloadValid": "{% $states.result.isInformationValid %}",
    "isValidAge": "{% $number($states.result.Age) >= 2 and $number($states.result.Age) < 5 %}"
    },
    "Next": "Information Check Result"
{% endraw %}
{% endhighlight %}

We store the original input data into variable of `$inputPayload` and the calculated age value into `$Age`. We store the output of the first data validation into `isPayloadValid` as a boolean value and we compute the age check and store is as a boolean value into `isValidAge`.

{% highlight json %}
{% raw %}
"Information Check Result": {
    "Type": "Choice",
    "Choices": [
    {
        "Condition": "{% $isPayloadValid and $isValidAge %}",
        "Next": "Check Spots Availability"
    },
    {
        "Condition": "{% $isValidAge = false %}",
        "Next": "Age Invalid"
    }
    ],
    "Default": "Notify Missing Info"
},
{% endraw %}
{% endhighlight %}

For the `Information Check Result` stage, we check if the data is valid by computing the boolean of the two previous stage variables. If its valid, we proceed to `Check Spots Availability` stage. If the age is invalid, we invoke the `Age Invalid` stage. If the payload is invalid, it invokes `Notify Missing Info` which is set as the default step.

For the `Check Spots Availability`, we retain the previous lambda. Refactoring it would require using a service such as DynamoDB to store the available number of vacancies:

{% highlight json %}
{% raw %}
 "Check Spots Availability": {
    "Type": "Task",
    "Resource": "arn:aws:states:::lambda:invoke",
    "Output": "{% $states.result.Payload %}",
    "Arguments": {
        "FunctionName": "arn:aws:lambda:eu-west-2:XXXXXXfunction:CheckSpotAvailable:$LATEST",
        "Payload": "{% $inputPayload %}"
    },
    "Assign": {
        "checkStatusCode": "{% $states.result.Payload.statusCode %}"
    },
    "Next": "Spot Availability Result"
},
{% endraw %}
{% endhighlight %}

Note that we are reusing the variable `$inputPayload` which was set at the beginning of the workflow.

The final stage `Spot Availability Result` would invoke `Add Registration` if there is sufficient capacity; if not it would invoke a fail stage with an error message:

{% highlight json %}
{% raw %}
"Spot Availability Result": {
    "Type": "Choice",
    "Choices": [
    {
        "Condition": "{% $checkStatusCode = 200 %}",
        "Next": "Add Registration"
    },
    {
        "Condition": "{% $checkStatusCode = 400 %}",
        "Next": "Notify No Spots"
    }
    ]
},
{% endraw %}
{% endhighlight %}

The `Add Registration` stage calls `dyanmoDB:putItem` to store the registration information into a DynamoDB table:

{% highlight json %}
{% raw %}
"Add Registration": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Arguments": {
        "TableName": "arn:aws:dynamodb:eu-west-2:XXXXX:table/TestRegistration",
        "Item": {
          "PK": {
            "S": "{% $uuid() %}"
          },
          "SK": {
            "S": "name"
          },
          "email": {
            "S": "{% $inputPayload.registration_info.parents.mother.email %}"
          },
          "name": {
            "S": "{% $inputPayload.registration_info.child.firstName & ' ' & $inputPayload.registration_info.child.lastName  %}"
          },
          "dateOfBirth": {
            "S": "{% $inputPayload.registration_info.child.dateOfBirth %}"
          },
          "age": {
            "N": "{% $string($Age) %}"
          },
          "specialInstructions": {
            "S": "{% $inputPayload.registration_info.specialInstructions %}"
          },
          "timestamp": {
            "S": "{% $now() %}"
          }
        }
      },
      "Next": "Confirm Registration"
    },
    "Confirm Registration": {
      "Type": "Succeed",
      "Comment": "Registration succeeded!"
    }
{% endraw %}
{% endhighlight %}

Note that we are using some of JSONata built-in functions such as `$uuid()` to generate a unique ID as primary key; `$now()` to obtain the current timestamp. JSONata variables allow us to transform data dynamically; in this example, we are concatenating the first and last name values. We are also reusing some of the previously assigned variables such as `$inputPayload` and `$Age` directly.

The complete state machine defintion is provided in the gist below:

<script src="https://gist.github.com/cheeyeo/d1882e476f63c594d8046a98fd934852.js"></script>

Note that, for the state machine service role, we would need to include additional permission policy to access DynamoDB.

While the removal of the data validation lambdas has simplied the workflow, it has also made the workflow difficult to test as we could have written pytest code for the lambda functions that were removed. For this example, I used the JSONata console at [https://try.jsonata.org] to formulate some of the inline functions before adding it to the state machine definition:

![JSONata console](/assets/img/aws/stepfunctions/jsonata_console.png)

It would be interesting to adopt a similar approach of using inline JSONata functions in future state machine workflows to see if it actually helps to simplify the development process.
