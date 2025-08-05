---
layout: post
show_meta: true
title: Invoke a SageMaker Inference endpoint using Lambda
header: Invoke a SageMaker Inference endpoint using Lambda
date: 2025-08-05 00:00:00
summary: How to invoke a SageMaker Inference endpoint using Lambda
categories: python aws lambda sagemaker triton
author: Chee Yeo
---

[Lambda HTTPS Endpoint]: https://aws.amazon.com/blogs/aws/announcing-aws-lambda-function-urls-built-in-https-endpoints-for-single-function-microservices/

[AWS Blog article]: https://aws.amazon.com/blogs/machine-learning/call-an-amazon-sagemaker-model-endpoint-using-amazon-api-gateway-and-aws-lambda/


In a previous post, we deployed a Triton Server container to SageMaker AI Inference Endpoint. The created endpoint has a HTTP URL of the form:

{% highlight shell %}
https://runtime.sagemaker.eu-west-2.amazonaws.com/endpoints/<ENDPOINT NAME>/invocations
{% endhighlight %}

However, this URL is not accessible via the public internet. It's only accessible from within the AWS Network. To test this deployed endpoint externally, we can create an API Gateway which forwards its requests to a Lambda function that makes a Triton client call to the inference endpoint as per this [AWS Blog article]. Since I would be using a Serverless Architecture with S3 events to invoke the endpoint and only need to test the endpoint for the time being, I adopted the same approach but using only a Lambda function.

The lambda function needs to have the **AWSLambdaBasicExecutionRole** as well as additional policy to call sagemaker invoke endpoint:

{% highlight json %}
 {
    "Effect": "Allow",
    "Action": [
        "sagemaker:InvokeEndpoint"
    ],
    "Resource": [
        "*"
    ]
}
{% endhighlight %}


The lambda function code is as follows:

{% highlight python %}
import os
import json
from urllib.parse import urlparse
import numpy as np
import tritonclient.http as httpclient
import boto3


SAGEMAKER_ENDPOINT = os.getenv("SAGEMAKER_ENDPOINT")
sagemaker_runtime = boto3.client("sagemaker-runtime", region_name='eu-west-2', verify=False)

def lambda_handler(event, context):
    img_path = event['image_path']
    np_input_data = np.asarray([img_path], dtype=np.object_)

    input_tensors = [
        httpclient.InferInput("ensemble_image_path", [1], "BYTES")

    ]
    input_tensors[0].set_data_from_numpy(np_input_data.reshape([1]))

    outputs = []
    outputs.append(httpclient.InferRequestedOutput("ensemble_output"))

    request_body, header_length = httpclient.InferenceServerClient.generate_request_body(
        input_tensors, 
        outputs
    )

    resp = sagemaker_runtime.invoke_endpoint(
        EndpointName=SAGEMAKER_ENDPOINT,
        ContentType="application/vnd.sagemaker-triton.binary+json;json-header-size={}".format(header_length),
        Body=request_body
    )

    # Parse json header size length from the response
    header_length_prefix = "application/vnd.sagemaker-triton.binary+json;json-header-size="
    header_length_str = resp["ContentType"][len(header_length_prefix):]

    # Read response body
    result = httpclient.InferenceServerClient.parse_response_body(
        resp["Body"].read(), header_length=int(header_length_str)
    )
    output0_data = result.as_numpy("ensemble_output").astype(str)
    print(output0_data)
{% endhighlight %}

The lambda function contains the logic to send a request to the inference endpoint. It expects the source image path as an input to the model. The image path in this case, is an S3 URI of the source image. This is converted into a numpy array and set as the data source using **httpclient.InferInput.set_data_from_numpy**. 

The ensemble model also returns a single string output of the destination S3 bucket of the converted image which we set in the outputs list using **httpclient.InferRequestedOutput**. Next, we create a sagemaker runtime client and invoke the endpoint using **sagemaker_runtime.invoke_endpoint** passing in as parameters the endpoint name, the request body and the content type set to **application/vnd.sagemaker-triton.binary+json**. 

On receiving the response, we extract the header length value and use it to parse the response body using **httpclient.InferenceServerClient.parse_response_body**. The response body is an object of type **botocore.response.StreamingBody**. We convert the result with the **ensemble_output** key and convert its value type from tensor into numpy and into a string output. If successful, the result should be the target s3 path of the converted image. 


The SageMaker endpoint logs below indicate that the ensemble model was invoked:

{% highlight shell %}
I0805 15:49:42.382778 93 sagemaker_server.cc:190] SageMaker request: 2 /invocations
I0805 15:49:42.382881 93 infer_request.cc:131] [request id: <id_unknown>] Setting state from INITIALIZED to INITIALIZED
I0805 15:49:42.382896 93 infer_request.cc:893] [request id: <id_unknown>] prepared: [0x0x7fd120003840] request id: , model: ensemble_model, requested version: -1, actual version: 1, flags: 0x0, correlation id: 0, batch size: 0, priority: 0, timeout (us): 0
original inputs:
[0x0x7fd1200144c8] input: ensemble_image_path, type: BYTES, original shape: [1], batch + shape: [1], shape: [1]
override inputs:
inputs:
[0x0x7fd1200144c8] input: ensemble_image_path, type: BYTES, original shape: [1], batch + shape: [1], shape: [1]

...

I0805 15:49:48.942384 93 http_server.cc:1199] HTTP using buffer for: 'ensemble_output', size: 43, addr: 0x7fd0c8005030
I0805 15:49:48.942390 93 pinned_memory_manager.cc:191] pinned memory deallocation: addr 0x7fd1848b89d0
I0805 15:49:48.942433 93 http_server.cc:1273] HTTP release: size 43, addr 0x7fd0c8005030
I0805 15:49:48.942475 93 infer_request.cc:131] [request id: <id_unknown>] Setting state from EXECUTING to RELEASED
I0805 15:49:48.942488 93 pinned_memory_manager.cc:191] pinned memory deallocation: addr 0x7fd1840000c0
I0805 15:49:48.942497 93 infer_request.cc:131] [request id: <id_unknown>] Setting state from EXECUTING to RELEASED
{% endhighlight %}


The output response from the lambda function logs show the response following the triton client call:

{% highlight shell %}
{'ResponseMetadata': {'RequestId': '6b9b9d11-b61c-4f0b-8fb3-6041d0f0ef2f', 'HTTPStatusCode': 200, 'HTTPHeaders': {'x-amzn-requestid': '6b9b9d11-b61c-4f0b-8fb3-6041d0f0ef2f', 'x-amzn-invoked-production-variant': 'AllTraffic', 'date': 'Tue, 05 Aug 2025 15:49:48 GMT', 'content-type': 'application/vnd.sagemaker-triton.binary+json;json-header-size=232', 'content-length': '275', 'connection': 'keep-alive'}, 'RetryAttempts': 0}, 'ContentType': 'application/vnd.sagemaker-triton.binary+json;json-header-size=232', 'InvokedProductionVariant': 'AllTraffic', 'Body': <botocore.response.StreamingBody object at 0x7f8a6ac08fa0>}
{% endhighlight %}

The screenshot below shows the contents of the target S3 bucket:

![Target S3 Bucket](/assets/img/triton/sagemaker/target_s3_bucket.png)

Using this approach, we are able to perform inference using Lambda. We can also make the Lambda function public by adding a [Lambda HTTPS Endpoint] or integrating it with an ALB. We also need to make the setup more secure by:

* Deploying the Sagemaker Endpoint into a private subnet in a VPC
* Deploying the Lambda function into a private subnet in a VPC

These will be explored further in future posts.
