---
layout: post
show_meta: true
title: Using Code Deploy and Github Actions to deploy ECS service
header: Using Code Deploy and Github Actions to deploy ECS service
date: 2024-06-10 00:00:00
summary: How to deploy an ECS service using Code Deploy and Github workflow
categories: aws ecs codedeploy terraform fastapi docker github-actions
author: Chee Yeo
---

[AWS ECR Login]: https://github.com/aws-actions/amazon-ecr-login

[AWS ECS DEPLOY TASK DEFINITION]: https://github.com/aws-actions/amazon-ecs-deploy-task-definition

[AWS ECS RENDER TASK DEFINITION]: https://github.com/aws-actions/amazon-ecs-render-task-definition


![Code Deploy Deployment](/assets/img/ecs/code_deploy_build.png)
![Github workflow](/assets/img/ecs/github_workflow_build.png)

In the previous article, I described a process of using terraform to provision the infrastructure required to run an ECS service. However, updating the service via Terraform was not ideal even in a development environment. I managed to come up with a process that uses AWS Code Deploy and GitHub actions to coordinate the development and deployment of the service, to create a smoother process.

Firstly, I removed the terraform modules for the task definition and created it as a single file in the repository **taskdef.json**. This will become clear in a moment when I describe the github actions I used in the workflow. AWS recommendation is to add the task definition into version control as this would allow us to track the lineage of the automated deployment i.e. which task definition created this service.

Next, I created a simple workflow for deployment which has 2 distinct jobs: build and deploy. The build stage will build the application into a Docker container and push it with its git commit SHA as its tag onto ECR. It will update the task definition **taskdef.json** with the image attribute pointing to the pushed container.

To achieve this, we would require the following open-source aws actions to interact with ECR:

* aws-actions/amazon-ecr-login
* aws-actions/amazon-ecs-render-task-definition

The **amazon-ecr-login** action allows the assumed IAM role to login to a private ECR repo. Following a successful build and push, the **amazon-ecs-render-task-definition** updates the **taskdef.json** file dynamically with the new image URL and returns a new task definition json file. We can save and upload this new task defintion as an artifact of the build stage for consumption in the deploy stage.

The workflow for the build stage now looks like this:

{% highlight yaml %}
permissions:
  id-token: write
  contents: read

jobs:
  build:
    name: Build Image
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        name: Checkout Repository

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.IAMROLE_GITHUB }}
          role-session-name: GitHub-Action-Role
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR Private
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: fastapi-dev
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG -f api_application/Dockerfile api_application

          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
      
      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: taskdef.json
          container-name: fastapi
          image: ${{ steps.build-image.outputs.image }}
      
      - name: Rename task definition
        run: |
          cp ${{ steps.task-def.outputs.task-definition }} taskdef-new.json
          cat taskdef-new.json
      
      - name: Upload Task Definition
        uses: actions/upload-artifact@v4
        with:
          name: taskdef
          path: taskdef-new.json
{% endhighlight %}

Under the step **task-def** we create and return an updated task definition. We renamed it as the step output returned a random filename. We create an artifact of it and upload it as **taskdef-new.json**. By versioning the task definition file, the build stage is possible.

For using Code Deploy, I created additional terraform modules which created an application and a deployment group:

{% highlight terraform %}
resource "aws_codedeploy_app" "example" {
  compute_platform = "ECS"
  name             = "ecs-demo-v2"
}

resource "aws_codedeploy_deployment_group" "example" {
    app_name = aws_codedeploy_app.example.name
    deployment_config_name = "CodeDeployDefault.ECSAllAtOnce"
    deployment_group_name = "ecs-demo-v2-dg"
    service_role_arn = "DevCodeDeployECSRole"

    auto_rollback_configuration {
        enabled = true
        events  = ["DEPLOYMENT_FAILURE"]
    }

    blue_green_deployment_config {
        deployment_ready_option {
            action_on_timeout = "CONTINUE_DEPLOYMENT"
        }

        terminate_blue_instances_on_deployment_success {
            action                           = "TERMINATE"
            termination_wait_time_in_minutes = 5
        }
    }

    deployment_style {
        deployment_option = "WITH_TRAFFIC_CONTROL"
        deployment_type   = "BLUE_GREEN"
    }

    ecs_service {
        cluster_name = aws_ecs_cluster.foo.name
        service_name = aws_ecs_service.fastapi_service.name
    }

    load_balancer_info {
        target_group_pair_info {
            prod_traffic_route {
                listener_arns = [aws_lb_listener.http.arn]
            }

            target_group {
                name = aws_lb_target_group.service_targetgroup_green.name
            }

            target_group {
                name = aws_lb_target_group.service_targetgroup_blue.name
            }
        }
    }
}
{% endhighlight %}

Within Code Deploy, every deployment belongs to a deployment group, which in turn belongs to an application. We create the application first using a **aws_codedeploy_app** resource. Next, we create it's associated deployment group using **aws_codedeploy_deployment_group**. It requires an IAM role with at least the **AWSCodeDeployRoleForECS** policy. You would need to add additional policies for S3 access if you are providing the appspec.json file via S3. 

By default, Code Deploy uses Blue-Green deployment for ECS services. This means we need to create an additional target group but we don't add this to the listener provisioned. 

Code Deploy will automatically:

* Create a replacement task set for the ECS service
* Register the new task to the second target group
* Deassociate the current target group from the ALB
* Associate the second target group to the ALB. 

This is specified in the *blue_green_deployment_config* above. 

The above can be replaced with an on-premise deployment strategy which links the Github repo via a Github token to the deployment group but it seems to require a running Code Deploy agent and only works for EC2 / on-prem instances. Future articles will explore how this setup works.

The **appspec.json** file is required for any Code Deploy deployment so it must be created and checked into version control. The basic structure is as so:

{% highlight json %}
{
    "version": 0.0,
    "Resources": [
      {
        "TargetService": {
          "Type": "AWS::ECS::Service",
          "Properties": {
            "TaskDefinition": "<TASK_DEFINITION>",
            "LoadBalancerInfo": {
              "ContainerName": "fastapi",
              "ContainerPort": 9000
            }
          }
        }
      }
    ]
  }
{% endhighlight %}

The Task definition attribute is set to a placeholder which will be replaced in the deploy stage.

The deploy job in the workflow is as follows:

{% highlight yaml %}
 deploy:
    name: Deploy
    runs-on: ubuntu-latest
    needs: build

    steps:
      - uses: actions/checkout@v4
        name: Checkout Repository
      
      - name: Download Task Definition
        uses: actions/download-artifact@v4
        with:
          name: taskdef
          path: taskdef
      
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.IAMROLE_GITHUB }}
          role-session-name: GitHub-Action-Role
          aws-region: ${{ env.AWS_REGION }}


      - name: Deploy to Amazon ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: taskdef/taskdef-new.json
          service: fastapi-service
          cluster: MyCluster
          wait-for-service-stability: true
          codedeploy-appspec: appspec.json
          codedeploy-application: ecs-demo-v2
          codedeploy-deployment-group: ecs-demo-v2-dg
{% endhighlight %}

We utilize the **amazon-ecs-deploy-task-definition** action provided by AWS. It takes as input the updated task defintion file from the build stage, which has now been downloaded and extracted into the working directory. It registers this task definition file provided as a new version under ECS.

The last three lines define the parameters of triggering a code deploy job. It requires the application and deployment group names. It also takes in the **appspec.json** file and replaces the task defintion placeholder with the newly registered task definition ARN from above. We set the job to wait until the service is successfully deployed but it's not necessary as we can check the deployment job status in the console UI.

Once deployed, we can access the service with the latest changes via ALB.

Further examples and usage of the Github Actions can be found at the following links: [AWS ECR Login], [AWS ECS DEPLOY TASK DEFINITION], [AWS ECS RENDER TASK DEFINITION].

Screenshots below show Code Deploy making a successful deployment and the call logs from Github Actions:

![Code Deploy Deployment](/assets/img/ecs/code_deploy_build.png)

![Code Deploy Switch over](/assets/img/ecs/code_deploy_switch.png)


The code base will be open source once refactoring is completed. 

H4PPY H4CK1NG !