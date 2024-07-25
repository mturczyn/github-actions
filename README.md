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

First, create Microsoft Entra Id application with AZ CLI command:
```
$result = az ad app create --display-name 'github-workflow'
```
It returns data in JSON - we need `appId` and `id` properties:
```
  "appId": "9d09fd9e-eac3-4c57-b9ca-c0034f8e93d5",
  "id": "9d09fd9e-eac3-4c57-b9ca-c0034f8e93d5",
```
Then use those values to execute command to create workload identity - below we use `id`
```
az ad app federated-credential create `
   --id $id `
   --parameters '{\"name\":\"github-workflow\",\"issuer\":\"https://token.actions.githubusercontent.com\",\"subject\":\"repo:$mturczyn/$github-actions:ref:refs/heads/main\",\"audiences\":[\"api://AzureADTokenExchange\"]}'
```
Create service principal in Azure:

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