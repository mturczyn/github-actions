# Introduction and Basic Information

GitHub stores its default workflows in a special directory `.github/workflows`.

## Workload Identity

When using Azure and GitHub, deployments are authenticated using workload identity.

GitHub uses an action named `azure/login` to log into Azure when running a workflow. In Azure, permission needs to be granted for the workflow to work with authentication tokens.

# Variables

To use variables throughout the entire workflow file, we can define them right below the `on` keyword as follows:
```yaml
env:
  AZURE_RESOURCEGROUP_NAME: gh-actions
  AZURE_WEBAPP_NAME: webapp-gh-actions
```
Then we can use it like this:
```yaml
${{ env.AZURE_RESOURCEGROUP_NAME }}
```
There are some default environment variables to use:
- `github.sha`: The identifier of the Git commit that triggered the workflow to execute.
- `github.run_number`: A unique number for each run of a particular workflow in a repository.

## Secrets in Variables

When secret values are used as variables, we can access them with:
```yaml
${{ secrets.NAME_OF_THE_SECRET }}
```

# Creating Workload Identity

Workload identities are a feature of Microsoft Entra ID, which is a global identity service. Many companies use Microsoft Entra ID, and each company is called a tenant.

By letting Microsoft Entra ID know about an application, we create an application registration in Microsoft Entra ID. An application registration represents the application in Microsoft Entra ID.

An application registration can have federated credentials associated with it. They enable supported services (such as GitHub) to use a Microsoft Entra ID application. This effectively means that Microsoft Entra ID and the supported service (like GitHub) trust each other - this trust is called federation.

To create a workload identity, first, create a Microsoft Entra ID application with the Azure CLI command:
```sh
$result = az ad app create --display-name 'github-workflow'
```
It returns data in JSON - we need the `appId` and `id` properties:
```json
"appId": "9d09fd9e-eac3-4c57-b9ca-c0034f8e93d5",
"id": "9d09fd9e-eac3-4c57-b9ca-c0034f8e93d5",
```
Then use those values to execute a command to create federated identity - below we use `$id`
```sh
az ad app federated-credential create `
   --id $id `
   --parameters '{\"name\":\"github-workflow-cred\",\"issuer\":\"https://token.actions.githubusercontent.com\",\"subject\":\"repo:mturczyn/github-actions:ref:refs/heads/main\",\"audiences\":[\"api://AzureADTokenExchange\"]}'
```
If we use GitHub environment, the subject would look like `repo:mturczyn/github-actions:environment:Prod` where `Prod` is the name of the GitHub environment.

If we wanted to give federated credentials to a workflow running as a pull request check, we would use this convention: `repo:mturczyn/github-actions:pull_request`.

Create a service principal in Azure for our created Microsoft Entra application:
```sh
az ad sp create --id $appId
```
Then we need to capture the `appId` of the above-created service principal (let's assume we have `$servicePrincipalId` with that value) and use it below to create a role assignment in Azure:
```sh
az role assignment create `
   --assignee $servicePrincipalId `
   --role Contributor `
   --scope '/subscriptions/6c031f3a-0aa5-480d-b2ec-272b24779509/resourceGroups/intrinsic-rg'
```

# Workflow Triggers

To define a trigger, we need to define the `on` section in the workflow, for example:
```yaml
on:
  push:
    branches:
      - main
```
The above defines that the workflow will run on every change (push) to the branch `main`.

To specify multiple branches to trigger the workflow, we can use multiple branch names and also the wildcard `**` (below, a push to a branch starting with `release/` or `main` branch would trigger the workflow):
```yaml
on:
  push:
    branches:
      - main
      - 'release/**'
```
To exclude a branch from triggers, we can use:
```yaml
on:
  push:
    branches-ignore:
      - 'feature/**'
```
or (note the `!` before the branch name - this tells to exclude this branch from triggering the pipeline):
```yaml
on:
  push:
    branches:
      - '!feature/**'
```
The latter approach gives more flexibility, as there can be either `branches` or `branches-ignore` defined.

## Pull Request Event Triggers

We can also specify to execute the workflow on pull request triggers in GitHub:
```yaml
# run on pull request creation
on: pull_request
```
or
```yaml
# run on pull request close
on:
  pull_request:
    types: [closed]
```
Workflows with such triggers can act, for example, as pull requests automated checks.

## Path Filter on Triggers

We can also define changes to what paths would trigger the workflow:
```yaml
on:
  push:
    paths:
      - 'deploy/**'
      - '!deploy/docs/**'
```
We can also use `paths-ignore`, which works in a similar manner to the `branches-ignore` keyword. However, we can't use `paths` and `paths-ignore` in the same trigger.

## Schedule Trigger

To run a workflow on a schedule, we can define the trigger as follows:
```yaml
on:
  schedule:
    - cron: '0 0 * * *'
```
In this example, `0 0 * * *` means run every day at midnight UTC.

## Concurrency Control

By default, GitHub Actions allows multiple instances of your workflow to run simultaneously. For example, multiple commits in a short time to the same branch would trigger multiple workflow executions.

To change that default behavior, we can use the `concurrency` keyword. It needs to be specified to a string that is consistent across all runs of the workflow. Usually, it's just a hardcoded string:
```yaml
concurrency: MyWorkflowWithLimitedConcurrency
```
If we want to limit concurrency for workflows triggered by pull requests but still want parallel workflow execution from checks from different pull requests, we could use:
```yaml
concurrency: ${{ github.event.number }}
```
`github.event.number` is the number associated with the pull request.

## Authenticating from Workflow

To request a token in a workflow, we need to add the `permissions` property:
```yaml
permissions:
  id-token: write
  contents: read
```

# Controlling Jobs Execution in Workflow

If one job can be executed only when another job finishes successfully, we can express this with `needs: jobName`, for example:
```yaml
name: learn-github-actions
on: [push]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Here is where we'd perform the validation steps."
  deployUS: 
    runs-on: windows-latest
    needs: validate
    steps:
      - run: echo "Here is where we'd perform the steps to deploy to the US region."
  deployEurope: 
    runs-on: ubuntu-latest
    needs: validate
    steps:
      - run: echo "Here is where we'd perform the steps to deploy to the European region."
```
If we need to rollback some changes when the pipeline fails, we can apply it using the `if` keyword:
```yaml
name: learn-github-actions
on: [push]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Here is where we'd perform the validation steps."
  deploy: 
    runs-on: windows-latest
    needs: validate
    steps:
      - run: echo "Here is where we'd perform the steps to deploy."
  rollback: 
    runs-on: ubuntu-latest
    needs: deploy
    if: ${{ failure() }}
    steps:
      - run: echo "Here is where we'd perform the steps to roll back a failure."
```

# Environments

In the GitHub web page, there is a possibility to create *environments* (similarly to Azure DevOps).

In a workflow, we reference the environment by `environment: Prod`.

## Rollback Jobs

To rollback changes we could use the following job:
```yaml
rollback: 
  runs-on: ubuntu-latest
  needs: smoke-test
  if: ${{ always() && needs.smoke-test.result == 'failure' }}
  steps:
  - run: |
      echo "Performing rollback steps..."
```

# Reusing Workflows

We can create reusable sections of workflow definitions in separate YAML files. Then the reused workflow is called the *called workflow* and the workflow that uses it is called the *caller workflow*.

To tell GitHub Actions that a workflow can be called by another workflow, we define its trigger as follows:
```yaml
on:
  workflow_call:
```
In the caller workflow, we refer to the called workflow as follows:
```yaml
jobs:
  job:
    uses: ./.github/workflow/script.yml
```
We can also pass parameters (inputs) to the called workflow. We need to define them in the called workflow:
```yaml
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
Then in the called workflow, we can reference input variables like this:
```yaml
jobs:
  say-hello:
    runs-on: ubuntu-latest
    steps:
    - run: |
        echo Hello ${{ inputs.environmentType }}!
```
In the caller workflow, we pass variables with the `with` keyword:
```yaml
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

# Output Variable Values Across Workflow

To define an output variable from one job, we need to define:
```yaml
  outputs:
    appServiceAppName: ${{ steps.deploy.outputs.appServiceAppName }}
```
where `deploy` is the `id` of the step (we also need to define that `id`). Full example:
```yaml
job1:
  runs-on: ubuntu-latest
  outputs:
    appServiceAppName: ${{ steps.deploy.outputs.appServiceAppName }}
  steps:
  - uses: actions/checkout@v3
  - uses: azure/login@v1
    name: Sign in to Azure
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  - uses: azure/arm-deploy@v1
    id: deploy
    name: Deploy Bicep file
    with:
      failOnStdErr: false
      deploymentName: ${{ github.run_number }}
      resourceGroupName: Playground1
      template: ./deploy/main.bicep
```
In a later job, we can reference this output variable with `needs.job1.outputs.appServiceAppName`, which requires defining also the `needs: job1` property. Full example:
```yaml
job2:
  needs: job1
  runs-on: ubuntu-latest
  steps:
  - run: |
      echo "${{needs.job1.outputs.appServiceAppName}}"
```

# Workflow Artifacts

*Workflow artifacts* provide a way to store files in GitHub Actions, and they're associated with the particular run of your workflow. You use the `actions/upload-artifact` workflow action to instruct GitHub Actions to upload a file or folder from the runner's file system as a workflow artifact:
```yaml
- name: Upload folder as a workflow artifact
  uses: actions/upload-artifact@v3
  with:
    name: my-artifact-name
    path: ./my-folder
```

Use the `actions/download-artifact` action to download all of the workflow artifacts:
```yaml
- uses: actions/download-artifact@v3
```
Or, specify an artifact name to download just a specific artifact:
```yaml
- uses: actions/download-artifact@v3
  with:
    name: my-artifact-name
```

# Ephemeral Environments

Sometimes we need short-lived environments, for example to create an Azure environment for each pull request to test it.

In order to do that, we need to define a workflow that would run as a pull request check. This workflow would create a new ephemeral environment related to that pull request.

After the pull request is merged, the ephemeral environment created for the pull request should be deleted along with all its resources.

Examples of workflows creating ephemeral environments on pull requests can be found in `provision-ephemeral-environments`.

# Template Spec, Modules, and Container Registries

When we publish a template spec from our own computer by using the Azure CLI, we use a command like the following:
```sh
az ts create --name StorageWithoutSAS --location westus3 --display-name "Storage account with SAS disabled" --description "This template spec creates a storage account, which is preconfigured to disable SAS authentication." --version 1 --template-file main.bicep
```
We can convert this Azure CLI command to a GitHub Actions step:
```yaml
- name: Publish template spec
  uses: azure/cli@v1
  with:
    inlineScript: |
      az ts create \
        --name StorageWithoutSAS \
        --location westus3 \
        --display-name "Storage account with SAS disabled" \
        --description "This template spec creates a storage account, which is preconfigured to disable SAS authentication." \
        --version 1 \
        --template-file main.bicep
```
The workflow uses the same process to publish the template spec that we would use ourselves.

Similarly, when we publish a Bicep module from our own computer by using the Azure CLI, we use a command like the following:
```sh
az bicep publish --file module.bicep --target 'br:toycompany.azurecr.io/mymodules/myqueue:2'
```
We can convert this Azure CLI command to a GitHub Actions step too:
```yaml
- name: Publish Bicep module
  uses: azure/cli@v1
  with:
    inlineScript: |
      az bicep publish \
        --file module.bicep \
        --target 'br:toycompany.azurecr.io/mymodules/myqueue:2'
```