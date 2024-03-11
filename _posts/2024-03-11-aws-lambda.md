---
layout: post
show_meta: true
title: On developing AWS Lambda functions locally
header: On developing AWS Lambda functions locally
date: 2024-03-11 00:00:00
summary: An automated approach to developing and testing AWS lambda functions locally
categories: aws python lambda docker docker-compose reactjs
author: Chee Yeo
---

[AWS Base Image for python]: https://docs.aws.amazon.com/lambda/latest/dg/python-image.html
[APIGateway Lambda integration]: https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html#api-gateway-simple-proxy-for-lambda-output-format
[APIGateway Lambda Errors]: https://docs.aws.amazon.com/apigateway/latest/developerguide/handle-errors-in-lambda-integration.html#handle-standard-errors-in-lambda-integration
[Docker compose watch]: https://docs.docker.com/compose/file-watch/


While working on improving my frontend skills and understanding how React works, I decided to create a single-page-app (SPA) which fetches github repositories via the Github API. When deployed, the SPA would invoke an **APIGateway** route which would invoke a Lambda function as its integration that calls the Github API and then returning the response to the SPA for rendering.

While I am aware other frameworks exist in the wild, such as the AWS SAM framework, I wanted to test if I could do it without having to manage additional external dependencies.

The first challenge I faced was developing the Lambda function locally. My initial approach was to deploy it and modifying the code via the AWS console. However, this approach became error prone. 

In order to streamline the development process, I decided that the following needs to happen:

* Being able to reload the lambda code and rebuilding the image on changes.

* Allow webapp to interact with the lambda container endpoint.


### Local development of Lambda code

AWS lambda supports various deployment methods, one of which allows you to create a Docker image and deploy it as a Lambda function. Using [AWS Base Image for python], I was able to build and deploy it seamelessly via the AWS CLI to ECR and then deploy to Lambda.

To enable dynamic reload of the lambda code, we can run the lambda container via the [Docker compose watch] command. We can run the lambda container as a service via docker compose:

{% highlight python %}
services:
  lambda:
    build:
      context: .
      dockerfile: ./deploy/lambda.dockerfile
    ports:
     - 9000:8080
    develop:
      watch:
        - action: rebuild
          path: ./lambda/lambda_function.py
{% endhighlight %}

We can then invoke the command as follows:
{% highlight python %}
  docker compose watch
{% endhighlight %}

The screenshot below shows the docker compose action running. 
![Running docker compose watch](/assets/img/lambda/docker_compose_watch1.png)

The second screenshot shows docker compose triggering a rebuild of the image when changes are detected in the lambda source code.

![Running docker compose watch on reload](/assets/img/lambda/docker_compose_watch2.png)

In the compose specification, the **develop** section initiates the watch action and configuration. The **action** specifies the action to take if the file object under **path** changes. In this instance, we are rebuilding and restarting the lambda container when changes are made to the lambda function file.

To tail the logs in a separate terminal, we can use the following command:
{% highlight python %}
   docker compose logs -f lambda_image
{% endhighlight %}

To invoke the lambda endpoint, we can use curl to send a POST request and check the response from the endpoint:
{% highlight python %}
curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{}'
{% endhighlight %}

> NOTE: The container endpoint requires a POST action to allow access. Using GET would cause the container to crash with a segfault.**


### ReactJS invocation of Lambda

The SPA uses the **Axios** library to call the lambda endpoint directly. However, this approach resulted in a segfault. 

After some testing of different approaches, I realised that I could proxy the request from the SPA client to the Lambda endpoint locally. To that end, I created a python Flask application to act as a proxy that accepts requests from the SPA and forwards it to the lambda container service.

The proxy application runs as a service via docker compose, using the watch handler. It specifies a link with the lambda service and passes the lambda function url as an environment variable:
{% highlight python %}
services:
  lambda:
    build:
      context: .
      dockerfile: ./deploy/lambda.dockerfile
    ports:
     - 9000:8080
    develop:
      watch:
        - action: rebuild
          path: ./lambda/lambda_function.py

  proxy:
    image: lambda-proxy
    build:
      context: .
      dockerfile: ./deploy/proxy.dockerfile
    ports:
      - 8000:8000
    links:
      - lambda:lambda
    depends_on:
      - lambda
    environment:
      LAMBDA_URL: http://lambda:8080/2015-03-31/functions/function/invocations
    develop:
      watch:
        - action: rebuild
          path: ./proxy/requirements.txt
        - action: rebuild
          path: ./proxy/application.py
{% endhighlight %}

The proxy application uses the **requests** library to call the lambda event handler via the container endpoint, including any query parameters. The response is formatted to match the response format of APIGateway. For example, the error response matches that of the [APIGateway Lambda Errors] response in order to synchronize the responses returned from both APIGateway and the local proxy without adding additional logic to the webapp.

By adopting this hybrid approach, I was able to develop, test and deploy the Lambda function as well as creating a reproducible environment that can be distributed in the form of Docker containers.

In future posts, I will be exploring other frameworks such as LocalStack and how they could help with the lambda development process.

The project referenced in this post can be accessed here:
https://github.com/cheeyeo/aws_lambda_docker

H4ppy H4ck1ng !!!