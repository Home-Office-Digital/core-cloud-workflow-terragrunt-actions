# core-cloud-workflow-terragrunt-actions

## Overview
This repository contains composite actions for individual terragrunt command fragments that you can use individually
- `terragrunt run-all init`
- `terragrunt hclfmt`
- `terragrunt run-all validate`
- `terragrunt run-all plan`
- `terragrunt run-all apply`

and a complete [Terragrunt workflow file](https://github.com/UKHomeOffice/core-cloud-workflow-terragrunt-actions/blob/main/.github/workflows/standard-pipeline.yml) when you just want to use a complete Terragrunt pipeline.

## Pre-requisites
1. Create a Github Environment.
2. Create a Github Environment secret called ACCOUNT_ID and add the AWS Account ID to it.

## Features
- If absent, the `terragrunt run-all init` fragment will bootstrap your defined state bucket and DynamoDB table for the relevant environment.
- A customised Github Actions summary page that provides at-a-glance debugging features and current configuration used without needing to delve into the steps or files.


## Usage of complete workflow file
    name: "Test Terragrunt Actions Pipeline"

    on:
      push:
        branches: [ main ]

    permissions:
      contents: read
      id-token: write
      actions: read
      security-events: write
      pull-requests: write

    jobs:
      <JOB_NAME>:
        uses: UKHomeOffice/core-cloud-workflow-terragrunt-actions/.github/workflows/standard-pipeline.yml@main
        with:
          github-environment: '<GITHUB_ENVIRONMENT_NAME>'
          state-bucket: '<DESIRED_NAME_OF_S3_BUCKET_FOR_STORING_STATE_FILES>'
          state-dynamodb-table: '<DESIRED_DYNAMODB_TABLE_NAME>'
          role-to-assume: '<AWS_ROLE_THAT_HAS_PERMISSIONS_TO_CREATE_AND_DESTROY_RESOURCES>'
          working-directory: '.'
        secrets:
          account_id: ${{ secrets.ACCOUNT_ID }}

Craete a workflow file in your `.github/workflows` directory and populate with the following, changing inputs and config as needed.

## Usage of composite actions
Please refer to the [Terragrunt workflow file](https://github.com/UKHomeOffice/core-cloud-workflow-terragrunt-actions/blob/main/.github/workflows/standard-pipeline.yml) for examples of using composite actions.
