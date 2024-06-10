---
layout: post
show_meta: true
title: Deploy FastAPI as a service to ECS 
header: How to deploy a service to ECS
date: 2024-06-03 00:00:00
summary: How to deploy a service to ECS
categories: aws ecs terraform fastapi docker
author: Chee Yeo
---

While learning and experimenting with FastAPI, I wanted to try to deploy it into a test environment in AWS. There are various ways of deploying a containerized application onto AWS:

* On Lambda as a docker container
* In an EKS cluster
* Using CodeDeploy service
* Using ECS 

Since my experience of ECS is minimal, I decided to create an ECS cluster and learn how to deploy an API service into a running cluster.

The process can be divided into the following stages:

* Creating an ECS cluster with a capacity provider
* Creating an ECS service
* Serving the ECS service via Application Load Balancer

The assumption here is that we are using an available private VPC with 2 public and 2 private subnets across multiple AZs. The ECS container instances and services will be deployed into private subnets without public IP associated. The Autoscaling strategy used is also minimal and simplistic and doesn't reflect production usage. 

We need to create a Launch Template first before we can create an Autoscaling Group to be used as capacity provider for the cluster. The Launch template needs to run a recent version of AMI that has both docker and the ECS agent installed. We can retrieve this AMI value via:

{% highlight shell %}
aws ssm get-parameters \ 
  --names /aws/service/ecs/optimized-ami/amazon-linux-2023/recommended \
  --region eu-west-1
{% endhighlight %}

Below is the terraform configuration of the Launch Template:
{% highlight terraform %}
resource "aws_launch_template" "ecs_runner" {
  name = "ECS_RUNNER"

  iam_instance_profile {
    name = "DevECSContainerInstanceRole"
  }

  image_id = "ami-XXXX"

  instance_initiated_shutdown_behavior = "terminate"

  instance_type = "t2.medium"

  monitoring {
    enabled = true
  }

  network_interfaces {
    associate_public_ip_address = false
    security_groups             = [aws_security_group.ecs_template.id]
    delete_on_termination       = true
  }


  tag_specifications {
    resource_type = "instance"

    tags = {
      Cluster = var.cluster_name
      Name    = "ContainerInstance"
    }
  }

  user_data = base64encode("${data.template_file.user_data.rendered}")
}
{% endhighlight %}

Within the Launch Template, we specify the AMI, the instance type, and the ECS container instance role. This is similar to the EC2 Instance role except it has the `AmazonEC2ContainerServiceforEC2Role` policy. I also added the `AmazonSSMManagedInstanceCore` policy for access via SSM since the instances will not have public IPs.

The launch template will also need to contain user data, which registers the ECS cluster name to the ecs agent config:

{% highlight shell %}
#!/bin/bash
echo ECS_CLUSTER=MyCluster >> /etc/ecs/ecs.config
{% endhighlight %}

We also define a security group which allows for both ports 9000 and 80 for the FastAPI service and the load balancer. The documentation states that when using a load balancer to route traffic to the service container running in the container instance, we need to define the inbound ports mapped from the running container to the host. In this case, the service runs on port 9000 and since ALB serves traffic on port 80, we add both of those as ingress rules to the SG.

{% highlight terraform %}
resource "aws_security_group" "ecs_template" {
  name        = "ECS-INSTANCE-SG"
  description = "SG for Container Instances"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 9000
    to_port     = 9000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
{% endhighlight %}

We associate the launch template to the autoscaling group. The ASG will launch new instances into the VPC private subnets with a min capacity of 1 and max capacity of 2:

{% highlight terraform %}
resource "aws_autoscaling_group" "ecs" {
  name             = "ECSAsg"
  desired_capacity = 1
  max_size         = 2
  min_size         = 1

  vpc_zone_identifier = module.vpc.private_subnets

  launch_template {
    id      = aws_launch_template.ecs_runner.id
    version = "$Latest"
  }

  tag {
    key                 = "AmazonECSManaged"
    value               = true
    propagate_at_launch = true
  }
}
{% endhighlight %}

When we create the cluster, we set the ASG as the default capacity provider. This is known as ECS Cluster Autoscaling. ECS creates and manages the EC2 container instances via a target tracking policy.

If it works, we should see the EC2 instance provisioned under `Infrastructure` tab in the cluster. You can test the ASG in the console by changing the desired, min and max values. This should affect the num of visible container instances registered in the cluster.

Once the container instances are running, we can work on creating the service. Note, there is no point in progressing further if there are no container instances since the service containers are running under docker on these instances.

Before we can create a service, we need to have a Task Definition. The basic object in an ECS cluster is a Task. Services are a form of long-running tasks associated with a deployment. 

Below is the Task Definition for the service in terraform:

{% highlight terraform %}
resource "aws_ecs_task_definition" "service" {
  family                   = "FASTAPI-DEV"
  requires_compatibilities = ["EC2"]
  network_mode             = "awsvpc"
  cpu                      = 1024
  memory                   = 512
  execution_role_arn       = "arn:aws:iam::XXXXXXX:role/DevECSTaskExecutionRole"
  runtime_platform {
    operating_system_family = "LINUX"
    cpu_architecture        = "X86_64"
  }
  container_definitions = <<TASK_DEFINITION
[
  {
            "name": "fastapi",
            "image": "XXXXXX.dkr.ecr.eu-west-1.amazonaws.com/fastapi-dev",
            "cpu": 0,
            "portMappings": [
                {
                    "name": "fastapi-9000-tcp",
                    "containerPort": 9000,
                    "hostPort": 9000,
                    "protocol": "tcp",
                    "appProtocol": "http"
                }
            ],
            "essential": true,
            "environment": [],
            "environmentFiles": [],
            "mountPoints": [],
            "volumesFrom": [],
            "ulimits": [],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/FASTAPI-DEV",
                    "awslogs-region": "${var.region}",
                    "awslogs-stream-prefix": "ecs"
                },
                "secretOptions": []
            },
            "systemControls": []
        }
]
  TASK_DEFINITION
}
{% endhighlight %}

The service container consists of a FastAPI service with its dependencies installed. We define a Task Execution Role which allows the ECS agent to pull the service container from ECR repository. This is not the same as a Task IAM role, which is assigned to the running task itself. Next, we define the cpu and memory usage of the service; how the service is to be run ( on EC2 hosts ); and the network mode of awsvpc.

Next, we define the Service:

{% highlight terraform %}
resource "aws_ecs_service" "fastapi_service" {
  name            = "fastapi-service"
  cluster         = aws_ecs_cluster.foo.id
  task_definition = aws_ecs_task_definition.service.arn
  desired_count   = 2

  network_configuration {
    subnets         = module.vpc.private_subnets
    security_groups = [aws_security_group.ecs_sg.id]
  }

  force_new_deployment = true
  
  placement_constraints {
    type = "distinctInstance"
  }

  capacity_provider_strategy {
    capacity_provider = aws_ecs_capacity_provider.capacity.name
    weight            = 100
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.service_targetgroup.arn
    container_name   = "fastapi"
    container_port   = 9000
  }

  depends_on = [aws_autoscaling_group.ecs]
}


resource "aws_security_group" "ecs_sg" {
  name        = "ECS-SERVICE-SG"
  description = "ECS Service Security Group"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port       = 0
    to_port         = 65535
    protocol        = "tcp"
    security_groups = ["${aws_security_group.ecs_alb.id}"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
{% endhighlight %}

The service references the task definition to use. By default, it fetches the latest version. The service is launched withn the private subnets of the VPC. It has a security group which references the ALB security group as its ingress source. We use the ASG attached to the cluster as the capacity provider, which means that the ASG can automatically provision container instances for the service containers based on its usage demands. We add the service to the target group used by the ALB, which adds the service container name and port to the ALB listener rules. All HTTP traffic to port 80 will redirect to port 9000 of the container instance.

The definition of the ALB and target group are as follows:
{% highlight terraform %}
resource "aws_security_group" "ecs_alb" {
  name        = "ApplicationLoadBalancerSG"
  description = "ALB Security Group"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}


resource "aws_lb_target_group" "service_targetgroup" {
  name        = "TestTargetGroup"
  port        = 80
  protocol    = "HTTP"
  target_type = "ip"
  vpc_id      = module.vpc.vpc_id

  health_check {
    path = "/"
  }
}

# Create Listener
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.service_alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.service_targetgroup.arn
  }
}

# Create application load balancer
resource "aws_lb" "service_alb" {
  name               = "TestALB"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.ecs_alb.id]
  subnets            = module.vpc.public_subnets

  tags = {
    Environment = "Development"
  }
}
{% endhighlight %}

Once it's deployed and the services are registered as healthy by the target group, we should be able to access it via the ALB public domain name ( A record ). The image below shows the Swagger documentation of the FastAPI service:

![Console UI for Swagger Docs](/assets/img/ecs/openapi_docs.png)

While it works, the setup can still benefit from some more refinements:

* Having a more fine-grained scaling policy, one for the cluster instances and another for the services. This would require the use of Application AutoScaling service. 

* Enable the deployment of the service to be dynamic, without the use of terraform. This could involve the use of a framework like AWS CDK which would create the service stack of ALB, target groups, and service. My initial efforts at installing the CDK resulted in a segfault so this was abandoned.

* Enable the use of TLS. This would require a certificate to be generated, the API service would need to run with SSL context enabled, the target group and ALB will need to be reworked to support HTTP2 with port 443.

While this has been a learning experience, I feel that the process of getting a containerized service to run in ECS is too complex for a development / test environment with too many moving parts especially with regards to setting up autoscaling ( though this could be mitigated by using Fargate launch type ? ) and application load balancers ( getting the security groups to work correctly )

In future post, I will be covering using other services such as CodeDeploy and compare it with this current setup.

H4PPY H4CK1NG !
