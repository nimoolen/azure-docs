---
title: Use a template to create a workspace
titleSuffix: Azure Machine Learning service
description: Learn how to use an Azure Resource Manager template to create a new Azure Machine Learning service workspace.
services: machine-learning
ms.service: machine-learning
ms.subservice: core
ms.topic: conceptual

ms.reviewer: larryfr
ms.author: haining
author: hning86
ms.date: 02/11/2019

# Customer intent: As a DevOps person, I need to automate or customize the creation of Azure Machine Learning service by using templates.
---

# Create an Azure Machine Learning service workspace by using a template

In this article, you learn several ways to create an Azure Machine Learning service workspace using Azure Resource Manager templates. 

For more information, see [Deploy an application with Azure Resource Manager template](../../azure-resource-manager/resource-group-template-deploy.md).

## Prerequisites

* An **Azure subscription**. If you do not have one, try the [free or paid version of Azure Machine Learning service](https://aka.ms/AMLFree).

* To use a template from a CLI, you need either [Azure PowerShell](https://docs.microsoft.com/powershell/azure/overview?view=azps-1.2.0) or the [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest).

## Resource Manager template

A Resource Manager template makes it easy to create resources as a single, coordinated operation. A template is a JSON document that defines the resources that are needed for a deployment. It may also specify deployment parameters. Parameters are used to provide input values when using the template.

The following template can be used to create an Azure Machine Learning service workspace and associated Azure resources:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "workspaceName": {
            "type": "string",
            "metadata": {
                "description": "The name of the Azure Machine Learning service workspace."
            }
        },
        "location": {
            "type": "string",
            "allowedValues": [
                "eastus",
                "eastus2",
                "southcentralus",
                "southeastasia",
                "westcentralus",
                "westeurope",
                "westus2"
            ],
            "metadata": {
                "description": "The location where the workspace will be created."
            }
        }
    },
    "variables": {
        "storageAccount": {
            "name": "[concat('sa',uniqueString(resourceGroup().id))]",
            "type": "Standard_LRS"
        },
        "keyVault": {
            "name": "[concat('kv',uniqueString(resourceGroup().id))]",
            "tenantId": "[subscription().tenantId]"
        },
        "applicationInsightsName": "[concat('ai',uniqueString(resourceGroup().id))]",
        "containerRegistryName": "[concat('cr',uniqueString(resourceGroup().id))]"
    },
    "resources": [
        {
            "name": "[variables('storageAccount').name]",
            "type": "Microsoft.Storage/storageAccounts",
            "location": "[parameters('location')]",
            "apiVersion": "2018-07-01",
            "sku": {
                "name": "[variables('storageAccount').type]"
            },
            "kind": "StorageV2",
            "properties": {
                "encryption": {
                    "services": {
                        "blob": {
                            "enabled": true
                        },
                        "file": {
                            "enabled": true
                        }
                    },
                    "keySource": "Microsoft.Storage"
                },
                "supportHttpsTrafficOnly": true
            }
        },
        {
            "name": "[variables('keyVault').name]",
            "type": "Microsoft.KeyVault/vaults",
            "apiVersion": "2018-02-14",
            "location": "[parameters('location')]",
            "properties": {
                "tenantId": "[variables('keyVault').tenantId]",
                "sku": {
                    "name": "standard",
                    "family": "A"
                },
                "accessPolicies": []
            }
        },
        {
            "name": "[variables('applicationInsightsName')]",
            "type": "Microsoft.Insights/components",
            "apiVersion": "2015-05-01",
            "location": "[if(or(equals(parameters('location'),'eastus2'),equals(parameters('location'),'westcentralus')),'southcentralus',parameters('location'))]",
            "kind": "web",
            "properties": {
                "Application_Type": "web"
            }
        },
        {
            "name": "[variables('containerRegistryName')]",
            "type": "Microsoft.ContainerRegistry/registries",
            "apiVersion": "2017-10-01",
            "location": "[parameters('location')]",
            "sku": {
                "name": "Standard"
            },
            "properties": {
                "adminUserEnabled": true
            }
        },
        {
            "name": "[parameters('workspaceName')]",
            "type": "Microsoft.MachineLearningServices/workspaces",
            "apiVersion": "2018-11-19",
            "location": "[parameters('location')]",
            "dependsOn": [
                "[variables('storageAccount').name]",
                "[variables('keyVault').name]",
                "[variables('applicationInsightsName')]",
                "[variables('containerRegistryName')]"
            ],
            "identity": {
                "type": "systemAssigned"
            },
            "properties": {
                "friendlyName": "[parameters('workspaceName')]",
                "keyVault": "[resourceId('Microsoft.KeyVault/vaults',variables('keyVault').name)]",
                "applicationInsights": "[resourceId('Microsoft.Insights/components',variables('applicationInsightsName'))]",
                "containerRegistry": "[resourceId('Microsoft.ContainerRegistry/registries',variables('containerRegistryName'))]",
                "storageAccount": "[resourceId('Microsoft.Storage/storageAccounts/',variables('storageAccount').name)]"
            }
        }
    ]
}
```

This template creates the following Azure services:

* Azure Resource Group
* Azure Storage Account
* Azure Key Vault
* Azure Application Insights
* Azure Container Registry
* Azure Machine Learning workspace

The resource group is the container that holds the services. The various services are required by the Azure Machine Learning workspace.

The example template has two parameters:

* The **location** where the resource group and services will be created.

    The template will use the location you select for most resources. The exception is the Application Insights service, which is not available in all of the locations that the other services are. If you select a location where it is not available, the service will be created in the South Central US location.

* The **workspace name**, which is the friendly name of the Azure Machine Learning workspace.

    The names of the other services are generated randomly.

For more information on templates, see the following articles:

* [Author Azure Resource Manager templates](../../azure-resource-manager/resource-group-authoring-templates.md)
* [Deploy an application with Azure Resource Manager templates](../../azure-resource-manager/resource-group-template-deploy.md)
* [Microsoft.MachineLearningServices resource types](https://docs.microsoft.com/azure/templates/microsoft.machinelearningservices/allversions)

## Use the Azure portal

1. Follow the steps in [Deploy resources from custom template](https://docs.microsoft.com/azure/azure-resource-manager/resource-group-template-deploy-portal#deploy-resources-from-custom-template). When you arrive at the __Edit template__ screen, paste in the template from this document.
1. Select __Save__ to use the template. Provide the following information and agree to the listed terms and conditions:

    * Subscription: Select the Azure subscription to use for these resources.
    * Resource group: Select or create a resource group to contain the services.
    * Workspace name: The name to use for the Azure Machine Learning workspace that will be created. The workspace name must be between 3 and 33 characters. It may only contain alphanumeric characters and '-'.
    * Location: Select the location where the resources will be created.

    ![The template parameters in the Azure portal](media/how-to-create-workspace-template/template-parameters.png)

For more information, see [Deploy resources from custom template](../../azure-resource-manager/resource-group-template-deploy-portal.md#deploy-resources-from-custom-template).

## Use Azure PowerShell

This example assumes that you have saved the template to a file named `azuredeploy.json` in the current directory:

```powershell
New-AzResourceGroup -Name examplegroup -Location "East US"
new-azresourcegroupdeployment -name exampledeployment `
  -resourcegroupname examplegroup -location "East US" `
  -templatefile .\azuredeploy.json -workspaceName "exampleworkspace"
```

For more information, see [Deploy resources with Resource Manager templates and Azure PowerShell](../../azure-resource-manager/resource-group-template-deploy.md) and [Deploy private Resource Manager template with SAS token and Azure PowerShell](../../azure-resource-manager/resource-manager-powershell-sas-token.md).

## Use Azure CLI

This example assumes that you have saved the template to a file named `azuredeploy.json` in the current directory:

```azurecli-interactive
az group create --name examplegroup --location "East US"
az group deployment create \
  --name exampledeployment \
  --resource-group examplegroup \
  --template-file azuredeploy.json \
  --parameters workspaceName=exampleworkspace
```

For more information, see [Deploy resources with Resource Manager templates and Azure CLI](../../azure-resource-manager/resource-group-template-deploy-cli.md) and [Deploy private Resource Manager template with SAS token and Azure CLI](../../azure-resource-manager/resource-manager-cli-sas-token.md).

## Next steps

* [Deploy resources with Resource Manager templates and Resource Manager REST API](../../azure-resource-manager/resource-group-template-deploy-rest.md).
* [Creating and deploying Azure resource groups through Visual Studio](../../azure-resource-manager/vs-azure-tools-resource-groups-deployment-projects-create-deploy.md).
