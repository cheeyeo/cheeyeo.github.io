---
layout: post
show_meta: true
title: Building Flask web application in Github Actions
header: Building Flask web application in Github Actions
date: 2025-08-29 00:00:00
summary: How to build a Flask web application using Github Actions
categories: python github-actions ci/cd flask
author: Chee Yeo
---

[AWS Application Deployment Pipeline Reference]: https://aws-samples.github.io/aws-deployment-pipeline-reference-architecture/application-pipeline/

[Reusable Github workflow]: https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows

In my previous post, I described a process of using an inline services block to run an external Postgresql database to run unit tests as part of the CI pipeline for a Flask web application in Github Action.

However, running unit tests is just a single aspect of a CI/CD pipeline. As documented by the [AWS Application Deployment Pipeline Reference], we also need to include additional stages such as code linting and formatting; secrets detection; and static security testing (SAST). This post will detail how I incorporated those additional stages into the Github pipeline before building and pushing the final docker image into ECR.

The diagram in [AWS Application Deployment Pipeline Reference] shows an additional `Local Development` stage which incorporates the same stages as in the build stage. These could be incorporated into the development workflow using precommit hooks, which we will explore in a future post.

The code sample below shows an example Github Actions workflow for a Flask application:

{% highlight yaml %}
name: Pull Request
on:
  pull_request:
    types: [opened, reopened, synchronize]

jobs:
    code-quality:
        runs-on: ubuntu-latest
        needs: [unit-tests]
        steps:
        - name: Checkout
            uses: actions/checkout@v4

        - name: Install Ruff
            uses: astral-sh/ruff-action@v3
        
        - name: Lint code
            run: ruff check --output-format=github --target-version=py312
        
        - name: Code formatting
            id: format
            run: |
            ruff format --check --diff --target-version=py312 > res.md || {
                echo "## Formatting Issues" >> $GITHUB_STEP_SUMMARY
                echo '```diff' >> $GITHUB_STEP_SUMMARY
                cat res.md >> $GITHUB_STEP_SUMMARY
                echo '```' >> $GITHUB_STEP_SUMMARY
            }
            continue-on-error: true

    secrets-detection:
        runs-on: ubuntu-latest
        needs: [unit-tests]
        permissions:
        pull-requests: write
        contents: read
        steps:
        - name: Checkout
            uses: actions/checkout@v4
            with:
            fetch-depth: 0

        - name: Secrets Scanning
            uses: gitleaks/gitleaks-action@v2
            env:
            GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
            GITLEAKS_ENABLE_COMMENTS: false

    security-testing:
        needs: [unit-tests]
        permissions:
        actions: read
        contents: read
        uses: ./.github/workflows/security_scan.yml
{% endhighlight %}

The jobs above correspond to the follow stages in the Build stage:
* Code Quality
* Secrets Detection
* Static Application Security Testing ( SAST )
* Package Artifacts


For Code Quality we are using `ruff` which is a Python linter and code formatter. The recommended approach is to also use it in pre-commit hooks for local development. In the job, we are installing the `astral-sh/ruff-action` action and then performing a linting and formatting check. For the formatting check, we output any errors found to the workflow summary.

For secrets detection, we are using the `gitleaks/gitleaks-action` action which scans the repository for any secrets commited such as API keys. 

For SAST, we created another workflow which uses `bandit`. `bandit` is a tool which scans a python file, builds an abstract syntax tree ( AST ) from it and applies the appropriate plugins during a scan to discover any security issues. 


The external workflow is created as a [Reusable Github workflow]. The workflow is registered as a `workflow_call` type which allows it to be imported and referenced in the calling workflow. The initial idea here is to create a reusable workflow which can be used later in other pipelines.

The steps for the security scan worflow is as follows:

{% highlight yaml %}
name: Bandit security tests

on:
  workflow_call:

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      
      - name: Setup python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12.11"
      
      - name: Install Bandit
        shell: bash
        run: pip install bandit[sarif,toml]

      - name: Perform Bandit Analysis
        shell: bash
        run: |
          bandit -c bandit.yml -r . -f sarif -o results.sarif || true
      
      - name: Upload SARIF file
        uses: actions/upload-artifact@v4
        with:
          name: bandit-results.sarif
          path: |
            results.sarif
{% endhighlight %}

The `bandit` program uses a configuration file in the project repository which specifies which directories to ignore as well as some checks to skip. The scan results are output into a `results.sarif` file which is attached as an artifact in the next step.

Under `Packaged Artifacts` we use docker to build the application image and deploy it to AWS ECR service. The image will be tagged with the SHA of the build. The build step is described below:

{% highlight yaml %}
  build:
    runs-on: ubuntu-latest
    outputs:
      image_uri: ${{ steps.ecr_build.outputs.image_uri }}
    permissions:
      id-token: write
      contents: read
    needs: [code-quality, secrets-detection, security-testing]
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@main
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/${{ env.AWS_GITHUB_ROLE }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Test AWS credentials
        run: |
          aws sts get-caller-identity
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build and push image
        id: ecr_build
        env:
          REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          REPOSITORY: flaskapp
        run: |
          SHORT_SHA=$(echo $GITHUB_SHA | cut -c1-7)
          docker build -t $REGISTRY/$REPOSITORY:$SHORT_SHA .
          docker push $REGISTRY/$REPOSITORY:$SHORT_SHA

          echo "image_uri=${REPOSITORY}:${SHORT_SHA}" >> "$GITHUB_OUTPUT"
{% endhighlight %}

The build stage relies on the previous three stages described above to complete successfully before it can proceed. We use the official `aws-actions/configure-aws-credentials` action to create short-lived dynamic credentials. The action requires a pre-configured IAM OIDC role which has the right permission policy to access the current github repository. We call `aws sts get-caller-identity` to check that the credentials were issued successfully. Next we use the official `aws-actions/amazon-ecr-login` to login to ECR. Finally, we call docker to create a build based on the current commit SHA and pushing it into the ECR repository. We also output the final ECR image URI as an output of the current stage so it can be reused in another part of the pipeline.

The screenshot below shows the pipeline running successfully on a push to a pull request branch:
![Github workflow run build stage](/assets/img/github/cicd/github_build_pipeline.png)

The screenshot below shows a successful ECR image build and push:
![ECR repository](/assets/img/github/cicd/ecr_repository.png)

The build pipeline shown here is still a works in progress. We still have not implemented the following steps:

* software component analysis scans using Dependabot or Renovate to check for vulnerabilities in dependencies

* software bill of materials ( SBOM ) which details all the dependencies used. 

Future posts will explore these areas as the pipeline is improved upon.
