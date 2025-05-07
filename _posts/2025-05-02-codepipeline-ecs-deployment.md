---
layout: post
show_meta: true
title: Using AWS CodePipeline to automate blue/green ECS deployment
header: Using AWS CodePipeline to automate blue/green ECS deployment
date: 2025-05-02 00:00:00
summary: Using AWS CodePipeline to automate blue/green ECS deployment
categories: aws codepipeline codebuild codedeploy CI/CD pipeline
author: Chee Yeo
---

To automate the deployment of an ECS service, we can utilise the features of CodePipeline to orchestrate a blue/green deployment. This post assumes the following pre-requisites:

* An ECS cluster with a running ECS service deployed to a VPC with at least 2 subnets and at least 2 AZs.
* An application load balancer running with at least 2 listeners and 2 target groups. One target group would be running the current taskset. The second is for the replacement taskset.

AWS CodePipeline allows you to build CI/CD workflows or pipelines running natively on AWS. It's a complicated product compared to other CI/CD solutions such as Github Actions as it comprises of 3 separate products: CodeBuild, CodeDeploy, CodePipeline.

Firstly, we create a CodePipeline project. The pipeline needs at least 2 stages to be valid. The first stage is the `Source` stage where we specify the code repository. In my use case, I created a new `Connections` under CodeDeploy which installed a github connector app to link to a private repo in Github. We reference this connection under the source stage and specify the repository name and branch to build on. This creates a webhook under the github repository which forwards any push events to CodePipeline.

![Dynamic task defintion](/assets/img/codepipeline/source.png)


CodeBuild is used for building the software artifacts in the Build stage. It involves creating a Build project with an input source ( Github Repository ) and produces output aartifacts ( binaries, ECR images ). The build configuration is specified via `buildspec.yml` file which specifies the dependencies, phases and artifacts fot the build. An example `buildspec.yml` file for building an ECS container Service is provided below:

{% highlight yaml %}
version: 0.2

env:
  shell: bash
  secrets-manager:
    AWS_ACCOUNT_ID: "code_example_test:AWS_ACCOUNT_ID"
    IMAGE_REPO_NAME: "code_example_test:IMAGE_REPO_NAME"
  exported-variables:
    - AWS_ACCOUNT_ID
    - IMAGE_REPO_NAME
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
  build:
    commands:
      - echo "Using exported variable REPOSITORY_URI=$REPOSITORY_URI"
      - echo Build started on `date`
      - echo Building the Docker image...
      - cd application
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
      - cd ../
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo "Writing image detail file..."
      - IMAGEHASH=$(aws ecr describe-images --repository-name $IMAGE_REPO_NAME --image-ids imageTag=latest | jq -r '.imageDetails[0].imageDigest')
      - printf '{"ImageURI":"%s@%s"}' "$REPOSITORY_URI" "$IMAGEHASH" > imageDetail.json
      - echo "Writing image definitions file..."
      - printf '[{"name":"%s","imageUri":"%s"}]' "$IMAGE_REPO_NAME" "$REPOSITORY_URI:$IMAGE_TAG" > imagedefinitions.json
artifacts:
  files:
  - imagedefinitions.json 
  - imageDetail.json
  - appspec.yaml
  - taskdef.json
{% endhighlight %}

[Image definitions file reference]: https://docs.aws.amazon.com/codepipeline/latest/userguide/file-reference.html

The main phases of a buildspec are:
* install => Installs any dependencies and sets up the build environment.

* pre-build => Commands to run before build starts. In our example, we login to ECR in order to deploy the final image. We also set up some environment variables for later stages such as the commit hash from the latest commit.

* build => The actual build process. In our example, we are building the docker image and tagging it with `latest` and the last commit hash.

* post-build => The built image is pushed to ECR. We also generate two artifacts dynamically with the pushed image URI: `imagedefinitions.json` and `imageDetails.json`. For deploying ECS services, we need to update the associated task definition for the service with the latest image URI. According to [Image definitions file reference], depending on how the ECS service is deployed, we require one of the two formats: `imagedefinitions.json` for a simple deployment; `imageDetails.json` for a blue-green deployment. 

Under `env`, I specified using SecretsManager secret to retrieve some values to be exported for downstream stages. This would require additional IAM permissions to be added to the CodeBuild service role. In addition, CodeBuild also supports reading values from SSM parameters.

`artifacts` outputs the listed files as a ZIP archive which is uploaded to an S3 bucket created by CodePipeline to be passed to the `Deploy` stage.

We could also run unit and security tests in the build stage but its outside the scope of this simple article.

The `Deploy` stage is handled via CodeDeploy, which is a separate service that supports continuous delivery. For this example, we created both the `CodeDeploy Application` and `CodeDeploy Deployment Group` before linking it via CodePipeline.

The configuration is handled by `appspec.yaml` which defines the ECS service we are deploying to:

{% highlight yaml %}
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "codepipeline-test"
          ContainerPort: 80
{% endhighlight %}

The placeholder `<TASK_DEFINITION>` will be replaced by the latest taskdefintion ARN. The second required file, `taskdef.json` contains the task definition which is used to create the service:

{% highlight json %}
{
    "executionRoleArn": "XXXXXX",
    "containerDefinitions": [
        {
            "name": "codepipeline-test",
            "image": "<IMAGE1_NAME>",
            "essential": true,
            "portMappings": [
                {
                    "hostPort": 80,
                    "protocol": "tcp",
                    "containerPort": 80
                }
            ]
        }
    ],
    "requiresCompatibilities": [
        "FARGATE"
    ],
    "networkMode": "awsvpc",
    "cpu": "256",
    "memory": "512",
    "family": "ecs-demo"
}
{% endhighlight %}

Note the placeholder `<IMAGE1_NAME>`. This will be replaced by the recently pushed ECR image value from `imageDetails.json` artifact from the Build stage. This file will have the following format after the build:

{% highlight json %}
{
"ImageURI": "ACCOUNTID.dkr.ecr.us-west-2.amazonaws.com/dk-image-repo@sha256:example3"
}
{% endhighlight %}

Under the `Dynamically update task definition image` section of the `Deploy` stage, we reference the `BuildArtifact` from the `Build` stage to use the JSON file to update the image URI: 

![Dynamic task defintion](/assets/img/codepipeline/codedeploy_stage.png)

To support blue/green deployment we need to specify the application load balancers and target groups in the deployment group configuration:

![Dynamic task defintion](/assets/img/codepipeline/deployment_group.png)

Note that for blue/green deployments, we specify that the traffic be routed immediately using `CodeDeployDefault.ECSAllAtOnce` with a wait time of 5 minutes in order to test the pipeline. In production usage, we will need to adjust this appropriately.

To test the pipeline, we can make an initial change to the application's background colour. The screenshot below shows the initial web application:

![Web App](/assets/img/codepipeline/webapplication.png)


After the changes are pushed, we should see the pipeline running:

![CodePipeline](/assets/img/codepipeline/pipeline.png)

We can also see the individual deployment under `CodeDeploy > Deployments`
![CodePipeline](/assets/img/codepipeline/deployment.png)

Note that we can see traffic being rerouted from the first target group to the second target group of the replacement task set. After 5 minutes, we should see the changes being made:

![ALB Resource Map](/assets/img/codepipeline/network_resource_map.png)

![Updated Web App](/assets/img/codepipeline/updated_webapp.png)

In future posts, I hope to explore more detailed usage of CodePipeline including running unit tests, manual approvals and cross account deployments.