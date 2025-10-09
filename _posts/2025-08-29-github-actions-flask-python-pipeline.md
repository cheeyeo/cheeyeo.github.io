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

In my previous post, I described a process of using an inline services block to run an external Postgresql database to run unit tests as part of the CI pipeline for a Flask web application in Github Action.

However, running unit tests is just a single aspect of a CI/CD pipeline. As documented by the [AWS Application Deployment Pipeline Reference], we also need to include additional stages such as code linting and formatting; secrets detection; and static security testing (SAST). This post will detail how I incorporated those additional stages into the Github pipeline before building and pushing the final docker image into ECR.

The diagram in [AWS Application Deployment Pipeline Reference] shows an additional `Local Development` stage which incorporates the same stages as in the build stage. These could be incorporated into the development workflow using precommit hooks, which we will explore in a future post.

The code sample below shows an example Github Actions workflow for a Flask application:

```
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

```

The jobs above correspond to the follow stages in the Build:
* Code Quality
* Secrets Detection
* Static Application Security Testing ( SAST )
* Package Artifacts



( TODO )

For Code Quality we are using `ruff` which is a Python linter and code formatter. The recommended approach is to also use it in pre-commit hooks for local development. In the job, we are installing the `astral-sh/ruff-action` and then performing a linting and formatting check. For the formatting check, we output any errors found to the workflow summary

For secrets detection, we are using the `gitleaks/gitleaks-action` action which scans the repository for any secrets such as API keys which could have been commited by accident.

For SAST, we created another workflow which uses `bandit`. The workflow iscreated as a `workflow_call` type which allows it to be imported in and this helps to cut down on the length and complexity of the build workflow...

The workflow for security scan:
```
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
```

( TODO: Describe what bandit does and how it works )

( TODO: Describe build and push to ECR )