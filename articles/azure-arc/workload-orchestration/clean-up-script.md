---
title: Clean-Up Script for Workload Orchestration
description: This article provides a clean-up script for Azure Arc workload orchestration.
author: SoniaLopezBravo
ms.author: sonialopez
ms.topic: install-set-up-deploy
ms.date: 07/01/2025
---

# Clean-up script for workload orchestration

This script is used to clean up resources in a specified Azure resource group. It provides options to selectively delete specific resource types. By default, no resources are deleted unless explicitly specified for safety.

For more information about deleting resources created with workload orchestration, see [Delete resources in workload orchestration](delete-resources.md).

## Prerequisites

Set up your environment for workload orchestration. If you haven't, go to [Prepare your environment for workload orchestration](initial-setup-environment.md) to set up the prerequisites.

> [!NOTE]
> You need to have the necessary permissions to delete resources in the specified resource group. For most cases, by default your alias will have permission.

## Update RBAC (only for ADO pipeline)

If you are using the clean-up script as a part of `RGCleanup` step via ADO pipeline for your private resource group, then you need to add **Contributor** permissions to your private resource group for the object ID `63a63b4c-a8d7-4aba-9d46-7dd032c7ce4e`.

### [Azure CLI](#tab/azcli)

```powershell
az role assignment create --assignee "63a63b4c-a8d7-4aba-9d46-7dd032c7ce4e" --role "Contributor" --scope "/subscriptions/<your subscription>/resourceGroups/<yourResourceGroupName>"
```

### [Azure portal](#tab/azportal)

1. Sign in to the [Azure portal](https://portal.azure.com/).
1. Go to your resource group.
1. Click on **Access control (IAM)**.
1. Click on **Add** and select **Add role assignment**.
1. Select **Contributor** role.
1. Assign access to **User, group, or service principal**.
1. Enter the object ID **63a63b4c-a8d7-4aba-9d46-7dd032c7ce4e**.
1. Click on **Next** and then **Review + assign**.

***

## Run the clean-up script 

In the [ZIP folder](https://github.com/microsoft/AEP/blob/main/content/en/docs/Configuration%20Manager%20(Public%20Preview)/Scripts%20for%20Onboarding/Configuration%20manager%20files.zip) you downloaded as part of [Prepare your environment for workload orchestration](initial-setup-environment.md), you can find the PowerShell script `RGCleanScript.ps1` that allows you to clean up resources in a specified Azure resource group. This script is useful for removing all resources created with workload orchestration, including sites, targets, configurations, schemas, and solutions.

You execute the script by running the following command in PowerShell, replacing `<YourResourceGroupName>` with the name of your resource group. You can also specify which resources to delete by using the optional parameters.

```powershell
# Clean only specific resources (safe by default - nothing is deleted unless explicitly specified)
.\RGCleanScript.ps1 -resourceGroupName <YourResourceGroupName> [-deleteSite $true] [-deleteTarget $true] 

# Clean all resources at once
.\RGCleanScript.ps1 -resourceGroupName <YourResourceGroupName> -deleteAll $true

# Clean all deployed instances
.\RGCleanScript.ps1 -resourceGroupName <YourResourceGroupName> [-deleteInstance $true]
```

The `RGCleanScript.ps1` script contains the following parameters, which you can set to customize the cleanup process:

| Parameter                  | Required/Optional | Type    | Description                                                                                      |
|----------------------------|-------------------|---------|--------------------------------------------------------------------------------------------------|
| `resourceGroupName`        | Required          | string  | The name of the resource group to clean.                                                         |
| `subscriptionId`           | Optional          | string  | Subscription ID for resources (For Microsoft.Edge). Defaults to the subscription shown by `az cli`. |
| `contextSubscriptionId`    | Optional          | string  | Subscription ID where context is present (For Microsoft.Edge). Defaults to `az cli subscription`.   |
| `contextResourceGroupName` | Optional          | string  | Resource group of the Context (For Microsoft.Edge).                    |
| `contextName`              | Optional          | string  | Name of the Context (For Microsoft.Edge).                      |
| `deleteSite`               | Optional          | bool    | Delete site resources. Default is `false`.                                                       |
| `deleteTarget`             | Optional          | bool    | Delete target resources. Default is `false`.                                                     |
| `deleteConfiguration`      | Optional          | bool    | Delete CM created configuration resources. Default is `false`.                                   |
| `deleteSchema`             | Optional          | bool    | Delete schema/dynamic schema resources. Default is `false`.                                      |
| `deleteConfigTemplate`     | Optional          | bool    | Delete user created config template resources. Default is `false`.                               |
| `deleteSolution`           | Optional          | bool    | Delete solution template resources. Default is `false`.                                          |
| `deleteInstance`           | Optional          | bool    | Delete application instances. Default is `false`.                                                |
| `deleteAks`                | Optional          | bool    | Delete AKS cluster resources. Default is `false`.                                                |
| `deleteManagedIdentity`    | Optional          | bool    | Delete managed identity resources. Default is `false`.                                           |
| `deleteMicrosoftEdge`      | Optional          | bool    | Delete Microsoft Edge resources. Default is `false`.                                             |
| `deleteAll`                | Optional          | bool    | Delete all resources (sets all delete parameters to true). Default is `false`.                   |

