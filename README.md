# Intro and basic info

Github stores its default workflows in special directory `.github\workflows`.

## Workload identity

When using Azure and GitHub, deployments are authenticated sing workload identity.

Github uses action named `azure/login` to lgoin into Azure, when running a workflow. In Azure, permission needs to be granted for workflow to wotk with authenticastion tokens.

# Variables

To use variables throughout entire workflow file we can define them right below `on` keyword as follows:
```
env:
  AZURE_RESOURCEGROUP_NAME: gh-actions
  AZURE_WEBAPP_NAME: webapp-gh-actions
```
Then we can use it
```
${{ env.AZURE_RESOURCEGROUP_NAME }}
```
There are some default environment variables to use:
- `github.sha`: The identifier of the Git commit that triggered the workflow to execute.
- `github.run_number`: A unique number for each run of a particular workflow in a repository

## Secrets in variables

When secret values are used as variables, we can access them with
```
${{ secrets.NAME_OF_THE_SECRET }}
```

# Creating workload identity

Workload identities are a feature of Microsoft Entra ID, which is a global identity service. Many companies use Microsoft Entra ID, and each company is called a tenant.

By letting know Microsoft Entra ID about application, we create application registration in Microsoft Entra ID. An application registration represents the application in Microsoft Entra ID.

An appliciton registration can have federated credentials associated with it. They enable supported services (such as GitHub) to use a Microsoft Entra ID application. This is effectively saying Microsoft Entra ID and supported service (like GitHub) to trust each other - this trust is called federation.

To create workload identity, first, create Microsoft Entra Id application with AZ CLI command:
```
$result = az ad app create --display-name 'github-workflow'
```
It returns data in JSON - we need `appId` and `id` properties:
```
  "appId": "9d09fd9e-eac3-4c57-b9ca-c0034f8e93d5",
  "id": "9d09fd9e-eac3-4c57-b9ca-c0034f8e93d5",
```
Then use those values to execute command to create federated identity - below we use `$id`
```
az ad app federated-credential create `
   --id $id `
   --parameters '{\"name\":\"github-workflow-cred\",\"issuer\":\"https://token.actions.githubusercontent.com\",\"subject\":\"repo:mturczyn/github-actions:ref:refs/heads/main\",\"audiences\":[\"api://AzureADTokenExchange\"]}'
```
If we use GitHub environment, subject would look like `repo:mturczyn/github-actions:environment:Prod` where `Prod` is name of GitHub environment.

Create service principal in Azure for our created Microsoft Entra application:
```
az ad sp create --id $appId
```
Then we need to capture `appId` of above (let's assume we have `$servicePrincipalId` with that value) created service principal and use it below to create role assignment in Azure:
```
az role assignment create `
   --assignee $servicePrincipalId `
   --role Contributor `
   --scope '/subscriptions/6c031f3a-0aa5-480d-b2ec-272b24779509/resourceGroups/intrinsic-rg'
```
# Workflow triggers

To define trigger we need to define `on` section on workflow, for example:
```
on:
  push:
    branches:
      - main
```
Above defines, that the workflow would run on every change (push) to branch `main`.

To specify multiple branches to trigger workflow, we can use multiple branches names and also wildcard `**` (below push to branch starting with `release/` or `main` branch would trigger workflow)
```
on:
  push:
    branches:
      - main
      - 'release/**'
```
To exclude branch from triggers, we can use:
```
on:
  push:
    branches-ignore:
      - 'feature/**'
```
or (note `!` before branch name - this tells to exclude this branch from triggering pipeline):
```
on:
  push:
    branches:
      - '!feature/**'
```
Latter approach gives more flexibility, as there can be either `branches` or `branches-ignore` defines.

## Path filter on triggers

We can also define changes to what paths would trigger the workflow:
```
on:
  push:
    paths:
      - 'deploy/**'
      - '!deploy/docs/**'
```
We can also use `paths-ignore`, which works in a similar manner to the `branches-ignore` keyword. However, we can't use `paths` and `paths-ignore` in the same trigger.

## Schedule trigger

To run workflow on schedule, we can define trigger as follows:'
```
on:
  schedule:
    - cron: '0 0 * * *'
```
In this example, `0 0 * * *` means run every day at midnight UTC.

## Concurrency control

By default, GitHub Actions allows multiple instance of your workflow to run simultaneously. For example multiple commits in short time to the same branch would trigger multiple workflow executions.

To change that default behaviour, we can use `concurrency` keyword. It needs to be specified to a string that is consistent across all runs of the workflow. Usually it's just hardcoded string:
```
concurrency: MyWorkflowWithLimitedConcurrency
```

## Authenticating from workflow

To request token in a workflow, we need to add `permissions` property:
```
permissions:
  id-token: write
  contents: read
```

# Controlling jobs execution in workflow

If one job can be executed only when other job finishes successfully, we can express this with `needs: jobName`, for example:
```
name: learn-github-actions
on: [push]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Here is where you'd perform the validation steps."
  deployUS: 
    runs-on: windows-latest
    needs: validate
    steps:
      - run: echo "Here is where you'd perform the steps to deploy to the US region."
  deployEurope: 
    runs-on: ubuntu-latest
    needs: validate
    steps:
      - run: echo "Here is where you'd perform the steps to deploy to the European region."
```
If we need to rollback some changes when the pipeline fails, we can apply it using `if` keyword:
```
name: learn-github-actions
on: [push]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Here is where you'd perform the validation steps."
  deploy: 
    runs-on: windows-latest
    needs: validate
    steps:
      - run: echo "Here is where you'd perform the steps to deploy."
  rollback: 
    runs-on: ubuntu-latest
    needs: deploy
    if: ${{ failure() }}
    steps:
      - run: echo "Here is where you'd perform the steps to roll back a failure."
```

# Environments

In GitHub web page there is possibility to create *environments* (similairly to Azure DevOps).

In workflow, we reference the environment by `environment: Prod`.

# Rollback jobs

To rollback changes we could use following job:
```
rollback: 
  runs-on: ubuntu-latest
  needs: smoke-test
  if: ${{ always() && needs.smoke-test.result == 'failure' }}
  steps:
  - run: |
      echo "Performing rollback steps..."
```

# Reusing workflows

We can create reusable sections of workflow definitions in separate YAML files. THen the reused workflow is called *called workflow* and workflow that uses it is called *caller workflow*.

To tell GitHub actions that workflow can be called by other workflow we define its trigger as follows:
```
on:
  workflow_call:
```
In the caller workflow, we refer to called workflow as follows:
```
jobs:
  job:
    uses: ./.github/workflow/script.yml
```
We can also pass parameters (inputs) to called workflow. We need to define them in called workflow:
```
on:
  workflow_call:
    inputs:
      environmentType:
        required: true
        type: string
    secrets:
      AZURE_CLIENT_ID:
        required: true
      AZURE_TENANT_ID:
        required: true
      AZURE_SUBSCRIPTION_ID:
        required: true
```
Then in called workflow we can reference input variables as such:
```
jobs:
  say-hello:
    runs-on: ubuntu-latest
    steps:
    - run: |
        echo Hello ${{ inputs.environmentType }}!
```
In caller workflow we pass variables with `with` keyword:
```
jobs:
  job-test:
    uses: ./.github/workflows/script.yml
    with:
      environmentType: Test
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID_TEST }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```