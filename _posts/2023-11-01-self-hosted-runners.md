---
layout: post
show_meta: true
title: Self-hosted runners and automated Docker build
header: Self-hosted runners and automated Docker build
date: 2023-11-06 00:00:00
summary: Running self-hosted runners to build and deploy docker containers from github actions
categories: computer-vision opencv github-actions self-hosted-runners terraform research
author: Chee Yeo
---

[OpenCV dockerhub]: https://hub.docker.com/repository/docker/m1l0/opencv/general
[Validate Github deliveries]: https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries
[How to install docker scout cli]: https://github.com/docker/scout-cli/

In order to test my own computer vision code in different versions of OpenCV, I started an open-source project whereby I would build and upload certain versions of the library into docker containers and upload them to [OpenCV dockerhub]

The process was manual and my last attempts to automate this via github actions failed for the following reasons:

* The build time takes too long due to the number of dependencies needed such as python. This incurs significant run time from the allocated budget of runner minutes from github monthly.

* The build didn't include safe practices such as running automated security scans on the buld artifacts to ensure it didn't contain any CVEs. 

* The build didn't include any tests on the artifacts. In some instances, built images which had faults were uploaded to dockerhub which meant more time spent rebuilding and debugging.

The following is the strategy I adopted to address the issues above.


### Using docker scout

The project uses `snyk` image scan for vulnerabilities during build. I switched over to using docker scout which is an additional plugin to install to the docker engine. Docker scout is bundled with docker desktop by default.

To install docker scout without docker desktop, I followed the following instructions listed on [How To install docker scout cli]. Once installed, we can test it locally via the following commands:
{% highlight shell %}
# To get list of CVEs
docker scout cves test:main --ignore-base --only-fixed

# Get list of recommendations
docker scout recommendations test:main
{% endhighlight %}

There's a github action that runs docker scout. However, if we are using self-hosted runners, we need to ensure docker scout is installed with the docker engine.
{% highlight yaml %}
...

- name: Docker Scout
  id: docker-scout
  uses: docker/scout-action@v0.18.1
  with:
    command: cves,recommendations
    image: m1l0/opencv:${{ matrix.opencv-version }}-python${{ matrix.python-version }}-ubuntu${{ matrix.ubuntu-version }}
    ignore-unchanged: true
    only-severities: critical,high
    write-comment: false
    exit-code: true
    summary: true

...
{% endhighlight %}


### Refactoring the Dockerfile

The Dockerfiles use the same set of shell scripts which install the core system packages that are required to build python. However it included the `software-properties-common` and `python3-dev` packages, which have older dependencies of `setuptools` that result in high CVEs during the docker scout scan. Removing these packages from the scripts fixes the CVEs. However, one of the build dependencies `lib-hdfs5` also resulted in an older version of `setuptools` being installed in the system python so an additional line of `pip --upgrade install setuptools` is required in the Dockerfile to remove further CVEs.


### Refactoring the github actions

The original github actions are triggered manually via a workflow dispatch action but as the versions of OpenCV increase, it becomes error prone to pass in the right versions of opencv and python to use for builds. Another issue was that it's impossible to have both CPU and GPU builds in a single workflow. Part of the refactoring involve splitting up the workflows into 2 separate entities for clarity.

Both workflows are similar expect for the build matrix of OS, opencv and python versions used. In the GPU workflow, there's an additional list of cuda versions.

The CPU dockerfile becomes:

{% highlight yaml %}
name: Docker build and push

on:
  workflow_dispatch:
  push:
    branches:
      - 'main'
    paths:
      - Dockerfile
      - scripts/*


permissions:
  contents: read
  packages: write
  pull-requests: write
  id-token: write

jobs:
  main:
    runs-on: self-hosted
    continue-on-error: true
    strategy:
      fail-fast: false
      matrix:
        ubuntu-version:
          # 23.10 breaks 4.5.5 
          # - '23.10'
          # 23.04 breaks 4.6.0 and 4.5.5
          # - '23.04'
          - 22.04
        opencv-version:
          - 4.8.0
          - 4.7.0
          - 4.6.0
          - 4.5.5
        python-version:
          - 3.11.6
          - 3.10.13

    steps:
      - name: Check out the repo
        uses: actions/checkout@v4

      # NOTE: Not working for self-hosted runners?
      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          driver-opts: network=host

      - name: Login to Dockerhub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Process python semvar
        id: pyvars
        run: |
          var=${{ matrix.python-version }}
          new_var="${var%.*}"
          new_var2="${new_var//./}"

          echo "PYVER: ${new_var}"
          echo "CPYTHON: ${new_var2}"

          echo "pyver=${new_var}" >> "$GITHUB_OUTPUT"
          echo "cpython=${new_var2}" >> "$GITHUB_OUTPUT"
        shell: bash

      - name: Build and export to Docker
        uses: docker/build-push-action@v5
        with:
          tags: m1l0/opencv:${{ matrix.opencv-version }}-python${{ matrix.python-version }}-ubuntu${{ matrix.ubuntu-version }}
          build-args: |
            PYVER=${{ steps.pyvars.outputs.pyver }}
            CPYTHON=${{ steps.pyvars.outputs.cpython }}
            PYTHON_VERSION=${{ matrix.python-version }}
            OPENCV_PACKAGE=opencv
            OPENCV_VERSION=${{ matrix.opencv-version }}
            UBUNTU=${{ matrix.ubuntu-version }}
          load: true # save to --output=type=docker
      
      - name: Inspect
        run: |
          docker image inspect m1l0/opencv:${{ matrix.opencv-version }}-python${{ matrix.python-version }}-ubuntu${{ matrix.ubuntu-version }}
        shell: bash
      
      - name: Run tests
        run: |
          docker run --rm -v ${{ github.workspace }}/tests:/tests m1l0/opencv:${{ matrix.opencv-version }}-python${{ matrix.python-version }}-ubuntu${{ matrix.ubuntu-version }} python /tests/test.py --image /tests/demo.jpg

          docker run --rm -v ${{ github.workspace }}/tests:/tests m1l0/opencv:${{ matrix.opencv-version }}-python${{ matrix.python-version }}-ubuntu${{ matrix.ubuntu-version }} /tests/build_cpp.sh
        shell: bash

      - name: Docker Scout
        id: docker-scout
        uses: docker/scout-action@v0.18.1
        with:
          command: cves,recommendations
          image: m1l0/opencv:${{ matrix.opencv-version }}-python${{ matrix.python-version }}-ubuntu${{ matrix.ubuntu-version }}
          ignore-unchanged: true
          only-severities: critical,high
          write-comment: false
          exit-code: true
          summary: true

      - name: Build and push to Dockerhub
        id: docker-build
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: m1l0/opencv:${{ matrix.opencv-version }}-python${{ matrix.python-version }}-ubuntu${{ matrix.ubuntu-version }}
          build-args: |
            PYVER=${{ steps.pyvars.outputs.pyver }}
            CPYTHON=${{ steps.pyvars.outputs.cpython }}
            PYTHON_VERSION=${{ matrix.python-version }}
            OPENCV_PACKAGE=opencv
            OPENCV_VERSION=${{ matrix.opencv-version }}
            UBUNTU=${{ matrix.ubuntu-version }}
{% endhighlight %}

The workflow gets triggered via workflow dispatch event or by merging into the main branch, if changes were made to the original Dockerfile and the build scripts.

Note that we specify using self-hosted runners under `runs-on: self-hosted`. We define the build matrix under `strategy` which defines the various opencv and python versions which are passed as build args to docker engine.

One of the additional action we implemented was to run a custom test suite after the image has been built but not pushed. The `docker build and push action` saves the built image on the self-hosted runner host but since `push: false` by default, the image is not sent to the dockerhub. We can then run the test suite on the built image, which will abort the build if any of the tests fail.

Next, we use the `docker scout action` to perform an initial scan of the built image. This is akin to running `docker scout cves --image <image id>` locally. We specify that we only want to capture issues which are critical and high and generate a report which gets attached to the workflow summary. We also specify the `exit_code` option which means that the step will fail with exit code 2 if any issues are detected.

Once the checks pass, the image will be uploaded to dockerhub.

The GPU dockerfile is similar except for the strategy matrix:
{% highlight yaml %}
...

    strategy:
      matrix:
        ubuntu-version:
          # NOTE: No cuda ubuntu images above 22.04 for now
          - 22.04
        opencv-version:
          - 4.8.0
          - 4.7.0
          - 4.6.0
          - 4.5.5
        python-version:
          - 3.11.6
          - 3.10.13
        cuda:
          - 11.8.0
          - 12.0.0
        cudnn:
          - 8
        # cuda support for 4.5. and 4.6.0 only up to cuda 11.8
        exclude: 
          - ubuntu-version: 22.04
            opencv-version: 4.5.5
            python-version: 3.11.6
            cuda: 12.0.0
            cudnn: 8
          - ubuntu-version: 22.04
            opencv-version: 4.5.5
            python-version: 3.10.13
            cuda: 12.0.0
            cudnn: 8
          - ubuntu-version: 22.04
            opencv-version: 4.6.0
            python-version: 3.11.6
            cuda: 12.0.0
            cudnn: 8
          - ubuntu-version: 22.04
            opencv-version: 4.6.0
            python-version: 3.10.13
            cuda: 12.0.0
            cudnn: 8
...
{% endhighlight %}

Note that we need two extra build args of cuda and cudnn to specify the version of CUDA and CUDNN libs to use for the build. We also have an `exclude` option which specifies that versions of OpenCV below 4.7.0 should not be built with CUDA 12.0.0. This is a known compatibility issue and by specifying it explicity using the exclude block, the workflow will only run the remaining combinations that don't include the stated combinations.


### On using self-hosted runners

Since each build takes a considerable time, its not feasible to rely on the default allowance of 2000 minutes for github workflows. 

To that end, I started creating self-hosted runners on EC2 instances. This would involve provisioning the instances and installing the github runner agent.

The following steps describe the process:

* Create a Personal Access Token (PAT) which has `workflow` and `admin:org` permissions to allow for the installation of custom runner scripts and subsequent API calls.

* Creating dynamic infrastructure on AWS for the self-hosted runners.

* Creating a webhook in the repo to forward all workflow events to a custom endpoint to create self-hosted runners.

For the PAT token, navigate to `Developer Settings > "Personal access tokens" > Tokens (classic)`. Then, click `Generate new token`. Provide a description and select the **workflow and admin:org** scopes. Note the secret value generated as this is required for SSM api calls below.


The required infrastructure to run self-hosted runners on AWS include:

* API gateway as endpoint for webhook.

* SSM agent installation on runner instances.

* CreateFleet to create EC2 instances on the fly.

* SSM RunCommand and parameters to run self-hosted runner install scripts from github.

* Custom scripts to install and uninstall runner agents.

* Custom lambdas to create and teardown the self-hosted runners.

* SQS queues to send github workflow events to lambda.

* IAM roles for the above.

To automate the process, I created a set of terraform modules which would create and teardown the required infrastructure above.


### How it works

After the webhook is registered, when a new workflow run is created, it sends an event of type `queue` to the lambda, which sends a SQS message to the work queue:

{% highlight python %}
body = json.loads(event['body'])
action = body['action']

if action == 'queued':
    sqs = boto3.client("sqs")
    resp = sqs.send_message(
        QueueUrl=os.environ.get('GITHUB_EVENTS_QUEUE'),
        MessageBody=event['body']
    )
    
    logger.debug(f'SQS MESSAGE ID: {resp["MessageId"]}')
        
    return {
        "isBase64Encoded": False,
        "statusCode": 200,
        "body": 'Runners created'
    }
{% endhighlight %}

The work queue sends the event to a second lambda which uses the information to create an EC2 instance using **FleetManager CreateFleet** api call. The api call includes information such as the capacity and the launch template to use:

{% highlight python %}
client = boto3.client('ec2')
resp = client.create_fleet(
    OnDemandOptions={
        'SingleInstanceType': True,
        'SingleAvailabilityZone': False,
        'MinTargetCapacity': 1
    },
    LaunchTemplateConfigs=[
        {
            'LaunchTemplateSpecification': {
                'LaunchTemplateName': 'GHRunnerCPU',
                'Version': '$Default'
            },
        }
    ],
    TargetCapacitySpecification={
        'TotalTargetCapacity': 1,
        'DefaultTargetCapacityType': 'on-demand'
    },
    Type='instant'
)


instance_id = resp['Instances'][0]['InstanceIds'][0]

if instance_id:
    ec2 = boto3.client('ec2')
    waiter = ec2.get_waiter('instance_status_ok')
    waiter.wait(
        InstanceIds=[instance_id]
    )
    
    print(f'Instance {instance_id} created and running')
    
    command_id = runSSMDocument(instance_id)
    logger.debug(f"Document Run: {command_id}")
    if waitForSSMDocumentToFinish(command_id, 90):
        logger.info("Command completed")
    else:
        logger.debug("Timeout or SSM Document failed! There is risk for a loose worker")

# Delete SQS message from queue
receipt_handle = event['Records'][0]['receiptHandle']
sqs = boto3.client("sqs")
resp = sqs.delete_message(
    QueueUrl=os.environ.get('SQS_QUEUE'),
    ReceiptHandle=receipt_handle
)
{% endhighlight %}

The launch template of **GHRunnerCPU** contains the AMI, instance type and configuration for each created instance including IAM roles to allow for SystemsManager access. Once the instance is created, we obtain its instance ID to run a custom script remotely via SSM to install the github runner agent. Once the remote command completes, we delete the message from the SQS queue to prevent duplicates.

The installed github runner agents are set to `ephermal` mode which means they are automatically deleted / removed as soon as the job is completed. This means we will have active / running EC2 instances without agents running. To that end, I created a lambda function that removes EC2 instances without a runner and EC2 Instances that have idle runners:

{% highlight python %}
def lambda_handler(event, context):
    runners = find_idle_runners()

    for runner in runners:
        command_id = runSSMDocument(runner)
        logger.debug(f"Document Run: {command_id}")

        if waitForSSMDocumentToFinish(command_id, runner, 90):
            logger.info("Command completed")
            terminate_instance(runner)
        else:
            logger.debug("Timeout or SSM Document failed! There is risk for a loose worker")
    
    # Check for running ec2 instances but without an agent installed
    running_ec2 = get_running_ec2()
    existing_runners = find_runners()

    for x in running_ec2:
        if x not in existing_runners:
            terminate_instance(x)


def get_running_ec2():
    client = boto3.client('ec2', region_name='eu-west-1')

    # Filter based on launch template id and state of running
    custom_filter = [
        {
            'Name': 'tag:aws:ec2launchtemplate:id',
            'Values': ['XXXX']
        },
        {
            'Name': 'instance-state-name',
            'Values': ['running']
        }
    ]

    instances = client.describe_instances(
        Filters=custom_filter
    )

    ids = list(map(lambda x: x['Instances'][0]['InstanceId'], instances['Reservations']))

    return ids


def find_runners():
    token = get_github_secret()
    
    url = "https://api.github.com/orgs/m1l0ai/actions/runners"
    header = {'Authorization': 'token {}'.format(token)}

    request = Request(
        method='GET',
        headers=header,
        url=url
    )

    with urlopen(request) as req:
        response = req.read().decode('utf-8')
        print(response)
        jsonData = json.loads(response)

    runners = []
    for runner in jsonData['runners']:
        runners.append(runner['name'].split("gh-runner-")[1])
    
    return runners


def find_idle_runners():
    token = get_github_secret()
    
    url = "https://api.github.com/orgs/m1l0ai/actions/runners"
    header = {'Authorization': 'token {}'.format(token)}

    request = Request(
        method='GET',
        headers=header,
        url=url
    )

    with urlopen(request) as req:
        response = req.read().decode('utf-8')
        print(response)
        jsonData = json.loads(response)

    runners = []
    for runner in jsonData['runners']:
        if runner['status'] == 'online' and runner['busy'] == False:
            runners.append(runner['name'].split("gh-runner-")[1])
    
    return runners
{% endhighlight %}

The token is the PAT token created earlier. Within the lambda handler, we first check for any idle runners, which are active EC2 instances with runners installed not doing any work. If so, we run an SSM Command on a custom script that stops and removes the runner, after which we terminate the instance via the `terminateInstance` function.

The next function call `get_running_ec2` tries to return a list of active EC2 instances created via the **CreateFleet** api by filtering on its launch template id in the instance tags. We then call `find_runners` to retrieve a list of active EC2 instances with runners working from github api. We compare the two lists and any instances not in the active list are deleted. 

The next issue to address is the lack of runners to perform any work if they are deleted when idle. I created a separate lambda function that runs periodically via a schedule to check that there are at least a minimum number of runners present:

{% highlight python %}
MAX_COUNT = 2

def lambda_handler(event, context):
    instances = get_running_ec2()

    if len(instances) < MAX_COUNT:
        sqs = boto3.client("sqs")
        resp = sqs.send_message(
            QueueUrl=QUEUE_URL,
            MessageBody=json.dumps({'workflow_run_id': None})
        )

    return {
        "isBase64Encoded": False,
        "statusCode": 200,
        "body": 'Runners created'
    }


def get_running_ec2():
    client = boto3.client('ec2', region_name='eu-west-1')

    # Filter based on launch template id and state of running
    custom_filter = [
        {
            'Name': 'tag:aws:ec2launchtemplate:id',
            'Values': ['XXXXXXX']
        },
        {
            'Name': 'instance-state-name',
            'Values': ['running']
        }
    ]

    instances = client.describe_instances(
        Filters=custom_filter
    )

    ids = list(map(lambda x: x['Instances'][0]['InstanceId'], instances['Reservations']))

    return ids

{% endhighlight %}

The lambda checks the number of running EC2 instances against an expected number. It sends a message to the SQS queue if the min number of runners are not met.

There will be a race condition between the lambda deleting EC2 instances and the lambda creating EC2 instances. In my use case, I set the schedule for the create lambda to run every 5 minutes and the delete lambda to run every 10 minutes. 

Once the infrastructure is setup, we need to use the API gateway URL in the webhook. I created aa repo level webhook rather than an organization webhook. To create a new webhook, we navigate to **Repo > Settings > Webhook**. Enter the API gateway URL in **Payload URL** field. Select **Content Type** to be **application/json**, add a secret value. Then **Enable SSL verification**. Under events, select only the **Workflow Runs** and **Workflow Jobs** events. Enable it by selecting **Active** and Save.

The secret token created for the webhook is used to [Validate Github deliveries]

Below is my implementation of the verification function. It uses SSM parameters to retrieve the stored github secret webhook token:

{% highlight python %}
def verify_signature(event):
    if 'x-hub-signature-256' not in event['headers'].keys():
        raise Exception("x-hub-signature-256 is missing!")
    
    client = boto3.client('ssm')

    try:
        response = client.get_parameter(
            Name=os.environ.get('GITHUB_SECRET'),
            WithDecryption=True
        )
        print(response)
    except Exception as e:
        print(e.response)

    github_secret = response["Parameter"]["Value"]
    github_signature = event['headers']['x-hub-signature-256']

    body = event.get('body', '')
    if event['isBase64Encoded'] == "true":
        body = base64.b64decode(body)

    hashsum = hmac.new(github_secret.encode('utf-8'), body.encode('utf-8'), digestmod=hashlib.sha256).hexdigest()
    
    expected_signature = f"sha256={hashsum}"
    
    if not hmac.compare_digest(expected_signature, github_signature):
        raise Exception("Request signatures don't match")
{% endhighlight %}

Once the webhook is enabled and working, each message is logged in the **deliveries tab** under webhooks > settings. I use this output to debug the webhook and its integration. We can also test the lambdas by using the output in the message body. The cloudwatch logs for the lambdas are an invaluable resource for triaging any issues.

With the above setup, I was able to train compute intensive docker builds which normally takes over an hour for each variant down to maximum of approximately 40 minutes. I was able to utilise the compute-optimized instances such as the **C7i** family with Intel processors to speed up the workloads.

In terms of costs, I paid per hour rather than github runner's pricing of per minute.

Given a single **c7i.4xlarge** instance @ **$0.76608 per hour** and 2 runners minimum, in the worse case scenario of each job running up to an hour, it would cost me **$1.60** per hour to run two jobs.


### Improvements

The following are issues I identified which might help improve the process:

* Create custom AMIs with all software preloaded including docker and github runner software to save provisioning time.

* Being able to run JIT runners via the Github API might save on running custom scripts remotely.

* Being able to create the infrastructure dynamically before running the workflow.

* How to resolve race condition between the create and delete runners lambdas.


