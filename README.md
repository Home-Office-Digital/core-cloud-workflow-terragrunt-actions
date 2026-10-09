# core-cloud-workflow-terragrunt-actions

## Overview
This repository contains composite actions for individual terragrunt command fragments that you can use individually
- `terragrunt run-all init`
- `terragrunt hclfmt`
- `terragrunt run-all validate`
- `terragrunt run-all plan`
- `terragrunt run-all apply`
- `terragrunt run-all plan -destroy` / `terragrunt run-all apply` for units listed in a `.deletions` manifest

and a complete [Terragrunt workflow file](https://github.com/Home-Office-Digital/core-cloud-workflow-terragrunt-actions/blob/main/.github/workflows/standard-pipeline.yml) when you just want to use a complete Terragrunt pipeline.

## Pre-requisites
1. Create a Github Environment.
2. Create a Github Environment secret called ACCOUNT_ID and add the AWS Account ID to it.

## Features
- If absent, the `terragrunt run-all init` fragment will bootstrap your defined state bucket and DynamoDB table for the relevant environment.
- A customised Github Actions summary page that provides at-a-glance debugging features and current configuration used without needing to delve into the steps or files.

  <img width="1192" height="1257" alt="Screenshot 2026-02-02 at 16 34 00" src="https://github.com/user-attachments/assets/7134a6af-d605-4a45-bf0e-4ce197eeb00a" />

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
        uses: Home-Office-Digital/core-cloud-workflow-terragrunt-actions/.github/workflows/standard-pipeline.yml@main
        with:
          github-environment: '<GITHUB_ENVIRONMENT_NAME>'
          state-bucket: '<DESIRED_NAME_OF_S3_BUCKET_FOR_STORING_STATE_FILES>'
          state-dynamodb-table: '<DESIRED_DYNAMODB_TABLE_NAME>'
          role-to-assume: '<AWS_ROLE_THAT_HAS_PERMISSIONS_TO_CREATE_AND_DESTROY_RESOURCES>'
          working-directory: '.'
        secrets:
          account_id: ${{ secrets.ACCOUNT_ID }}

Craete a workflow file in your `.github/workflows` directory and populate with the following, changing inputs and config as needed.

## Deleting resources
Deleting a resource's `terragrunt.hcl` file doesn't work as `run-all` only visits directories that still exist, so a removed file would just silently skip over it, rather than deleting resources.

Instead, add the full path of the resource you want destroyed to a `.deletions` file at the root of the environment directory, and leave the resource's `terragrunt.hcl` in place:

```bash
echo "terraform/environment/sandbox-ops-tooling/rds/test-terragrunt-1" > terraform/environment/sandbox-ops-tooling/.deletions
```

Once a resource has been destroyed, remove its directory and its entry in `.deletions` in a follow-up commit - the pipeline does not do this for you.

Note: `.deletions` only destroys whole resources. This is exactly what you want when a resource holds a single resource. If a module manages several resources, e.g. the [core-cloud-rds-tf-module](https://github.com/Home-Office-Digital/core-cloud-rds-tf-module) can hold several instances under one `instances` map), remove one entry from the map can be destroyed via a normal plan-apply, as the `terragrunt.hcl` file will still exist. If it's just got the one instance, then just add to the `.deletions` file like normal.

## Usage of composite actions
Please refer to the [Terragrunt workflow file](https://github.com/Home-Office-Digital/core-cloud-workflow-terragrunt-actions/blob/main/.github/workflows/standard-pipeline.yml) for examples of using composite actions.

## Additional configuration
The 'terragrunt-parallelism' input can be used to restrict the number of concurrent operations that the runner will execute. Reduce if your pipeline becomes unstable due to resource exhaustion.

The 'runner' input can be used to run the pipeline job on a specific runner label (e.g. a self-hosted or larger GitHub-hosted runner) instead of the default `ubuntu-latest`.
