---
title: Use Azure Devops CLI to manage variables in a variable group
description: Use this Azure DevOps CLI sample to create and manage secret and nonsecret variables in an Azure Pipelines variable group.
author: juliakm
ms.author: jukullam
manager: mijacobs
ms.date: 07/03/2024
ms.topic: sample
ms.devlang: azurecli 
monikerRange: '>=azure-devops-2020'
ms.custom: devx-track-azurecli
---

# Use Azure Devops CLI to manage variables in a variable group

[!INCLUDE [version-gt-eq-2020](../../../includes/version-gt-eq-2020.md)]

This sample uses the `az devops` Azure DevOps extension to the Azure CLI to create and run an Azure pipeline that accesses a variable group containing both secret and nonsecret variables.

The script demonstrates the following three operations:

- Defines an [Azure Pipelines](../../index.yml) pipeline by using a [YAML](/azure/devops/pipelines/yaml-schema/) file stored in GitHub.
- Creates a [variable group](../../library/variable-groups.md) with nonsecret and secret variables for use in the pipeline.
- Runs the pipeline using the [Azure DevOps CLI](../../../cli/index.md), and show pipeline run processing and output.

[!INCLUDE [include](~/../docs/reusable-content/azure-cli/azure-cli-prepare-your-environment.md)]

To run the sample, you need:

- A [GitHub repository with Azure Pipelines installed](../../get-started/pipelines-sign-up.md)
- A [GitHub personal access token (PAT)](https://docs.github.com/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) for access
- An [Azure DevOps organization](../../get-started/pipelines-sign-up.md) with a [personal access token (PAT)](../../../organizations/accounts/use-personal-access-tokens-to-authenticate.md#create-a-pat) for authentication
- [Project Collection Administrator permissions in the Azure DevOps organization](../../../organizations/security/change-organization-collection-level-permissions.md)

## Save the sample file

Save the following YAML pipeline definition as a file called *azure-pipelines.yml* in the root directory and `main` branch of your GitHub repository.

```yaml
parameters:
- name: image
  displayName: 'Pool image'
  default: ubuntu-latest
  values:
  - windows-latest
  - windows-latest
  - ubuntu-latest
  - ubuntu-latest
  - macOS-latest
  - macOS-latest
- name: test
  displayName: Run Tests?
  type: boolean
  default: false

variables:
- group: "Contoso Variable Group"
- name: va
  value: $[variables.a]
- name: vb
  value: $[variables.b]
- name: vcontososecret
  value: $[variables.contososecret]

trigger:
- main

pool:
  vmImage: ubuntu-latest

steps:
- script: |
    echo "Hello, world!"
    echo "Pool image: ${{ parameters.image }}"
    echo "Run tests? ${{ parameters.test }}"
  displayName: 'Show runtime parameter values'

- script: |
    echo "a=$(va)"
    echo "b=$(vb)"
    echo "contososecret=$(vcontososecret)"
    echo
    echo "Count up to the value of the variable group's nonsecret variable *a*:"
    for number in {1..$(va)}
    do
        echo "$number"
    done
    echo "Count up to the value of the variable group's nonsecret variable *b*:"
    for number in {1..$(vb)}
    do
        echo "$number"
    done
    echo "Count up to the value of the variable group's secret variable *contososecret*:"
    for number in {1..$(vcontososecret)}
    do
        echo "$number"
    done
  displayName: 'Test variable group variables (secret and nonsecret)'
  env:
    SYSTEM_ACCESSTOKEN: $(System.AccessToken)
```

## Run the sample

After you store the YAML file in GitHub, run the following Azure DevOps CLI script in a Bash shell in Cloud Shell or locally. The script creates the pipeline, variable group, and secret and nonsecret variables, and then modifies the variable values.

Before you run the script, replace the following placeholders as follows:

- `<devops-organization>` Your Azure DevOps organization name
- `<github-organization>` Your GitHub organization name
- `<github-repository>` Your GitHub repository name
- `<github-pat>` Your GitHub PAT

Replace the following placeholders with values you choose:

- `<pipelinename>` A name for the pipeline that is between 3-19 characters and contains only numerals and lowercase letters. The script adds a 5-digit unique identifier.
- `<azure-resource-group-location>` Azure region for the resource group that contains the Azure Storage account, for example `eastus`.
- `<azure-storage-account-location>` Same as the resource group location.

```azurecli
#!/bin/bash

# Provide placeholder variables.
devopsOrg="https://dev.azure.com/<devops-organization>"
githubOrg="<github-organization>"
githubRepo="<github-repository>"
githubPat="<github-pat>"
pipelineName="<pipelinename>"
resourceGroupLocation="<azure-resource-group-location>"
storageAccountLocation="<azure-storage-account-location>"
repoName="$githubOrg/$githubRepo"
repoType="github"
branch="main"

# Declare other variables.
uniqueId=$RANDOM
resourceGroupName="$pipelineName$uniqueId"
storageAccountName="$pipelineName$uniqueId"
devopsProject="Contoso DevOps Project $uniqueId"
serviceConnectionName="Contoso Service Connection $uniqueId"
variableGroupName="Contoso Variable Group"

# Sign in to Azure CLI and follow the sign-in instructions, if necessary.
echo "Sign in."
az login

# Sign in to Azure DevOps with your Azure DevOps PAT, if necessary.
echo "Sign in to Azure DevOps."
az devops login

# Create a resource group and a storage account.
az group create --name "$resourceGroupName" --location "$resourceGroupLocation"
az storage account create --name "$storageAccountName" \
    --resource-group "$resourceGroupName" --location "$storageAccountLocation"

# Set the environment variable used for Azure DevOps token authentication.
export AZURE_DEVOPS_EXT_GITHUB_PAT=$githubPat

# Create the Azure DevOps project and set defaults.
projectId=$(az devops project create \
    --name "$devopsProject" --organization "$devopsOrg" --visibility private --query id)
projectId=${projectId:1:-1}  # Just set to GUID; drop enclosing quotes.
az devops configure --defaults organization="$devopsOrg" project="$devopsProject"
pipelineRunUrlPrefix="$devopsOrg/$projectId/_build/results?buildId="

# Create GitHub service connection.
githubServiceEndpointId=$(az devops service-endpoint github create \
    --name "$serviceConnectionName" --github-url "https://www.github.com/$repoName" --query id)
githubServiceEndpointId=${githubServiceEndpointId:1:-1}  # Just set to GUID; drop enclosing quotes.

# Create the pipeline.
pipelineId=$(az pipelines create \
    --name "$pipelineName" \
    --skip-first-run \
    --repository $repoName \
    --repository-type $repoType \
    --branch $branch \
    --service-connection $githubServiceEndpointId \
    --yml-path azure-pipelines.yml \
    --query id)

# Create a variable group with 2 non-secret variables and 1 secret variable.
# (contososecret < a < b). Then run the pipeline.
variableGroupId=$(az pipelines variable-group create \
    --name "$variableGroupName" --authorize true --variables a=12 b=29 --query id)
az pipelines variable-group variable create \
    --group-id $variableGroupId --name contososecret --secret true --value 17
pipelineRunId1=$(az pipelines run --id $pipelineId --open --query id)
echo "Go to the pipeline run's web page to view the output results for the 1st run."
echo "If the web page doesn't automatically appear, go to:"
echo "    ${pipelineRunUrlPrefix}${pipelineRunId1}"
read -p "Press Enter to change the value of one of the variable group's nonsecret variables, then run again:"

# Change the value of one of the variable group's nonsecret variables.
az pipelines variable-group variable update \
    --group-id $variableGroupId --name a --value 22
pipelineRunId2=$(az pipelines run --id $pipelineId --open --query id)
echo "Go to the pipeline run's web page to view the output results for the 2nd run."
echo "If the web page doesn't automatically appear, go to:"
echo "    ${pipelineRunUrlPrefix}${pipelineRunId2}"
read -p "Press Enter to change the value of the variable group's secret variable, then run once more:"

# Change the value of the variable group's secret variable.
az pipelines variable-group variable update \
    --group-id $variableGroupId --name contososecret --value 35
pipelineRunId3=$(az pipelines run --id $pipelineId --open --query id)
echo "Go to the pipeline run's web page to view the output results for the 3rd run."
echo "If the web page doesn't automatically appear, go to:"
echo "    ${pipelineRunUrlPrefix}${pipelineRunId3}"
read -p "Press Enter to continue:"
```

## Clean up resources

To avoid incurring Azure storage charges after you run the script sample, you can remove the Azure resource group and all its resources by running the following script:

```azurecli
az pipelines variable-group delete --group-id $variableGroupId --yes
az pipelines delete --id $pipelineId --yes
az devops service-endpoint delete --id $githubServiceEndpointId --yes
az devops project delete --id $projectId --yes
export AZURE_DEVOPS_EXT_GITHUB_PAT=""
az storage account delete --name $storageAccountName --resource-group $resourceGroupName --yes
az group delete --name $resourceGroupName --yes
az devops configure --defaults organization="" project=""
```

## Azure CLI references

The sample in this article uses the following Azure CLI commands:

- [az devops configure](/cli/azure/devops#az-devops-configure)
- [az devops project create](/cli/azure/devops/project#az-devops-project-create)
- [az devops project delete](/cli/azure/devops/project#az-devops-project-delete)
- [az devops service-endpoint github create](/cli/azure/devops/service-endpoint/github#az-devops-service-endpoint-github-create)
- [az group create](/cli/azure/group#az-group-create)
- [az group delete](/cli/azure/group#az-group-delete)
- [az login](/cli/azure/reference-index#az-login)
- [az pipelines create](/cli/azure/pipelines#az-pipelines-create)
- [az pipelines delete](/cli/azure/pipelines#az-pipelines-delete)
- [az pipelines run](/cli/azure/pipelines#az-pipelines-run)
- [az pipelines variable-group create](/cli/azure/pipelines/variable-group#az-pipelines-variable-group-create)
- [az pipelines variable-group delete](/cli/azure/pipelines/variable-group#az-pipelines-variable-group-delete)
- [az pipelines variable-group variable create](/cli/azure/pipelines/variable-group/variable#az-pipelines-variable-group-variable-create)
- [az pipelines variable-group variable update](/cli/azure/pipelines/variable-group/variable#az-pipelines-variable-group-variable-update)
- [az storage account create](/cli/azure/storage/account#az-storage-account-create)
- [az storage account delete](/cli/azure/storage/account#az-storage-account-delete)
