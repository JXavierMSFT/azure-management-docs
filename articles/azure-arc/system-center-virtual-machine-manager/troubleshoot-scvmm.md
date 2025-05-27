---
title: Troubleshoot SCVMM-specific Azure Arc resource bridge deployment errors
description: Learn how to troubleshoot SCVMM-specific Azure Arc resource bridge deployment errors. 
ms.service: azure-arc
ms.subservice: azure-arc-scvmm
ms.author: jsuri
author: jyothisuri
ms.topic: how-to 
ms.date: 02/21/2025
keywords: "VMM, Arc, Azure, System Center"

# Customer intent: As an IT administrator, I want to resolve Azure Arc resource bridge deployment errors in SCVMM, so that I can successfully onboard resources to Azure and ensure seamless operations.
---

# Troubleshoot SCVMM-specific Azure Arc resource bridge deployment errors

This article provides troubleshooting steps that help you resolve the errors encountered during the deployment of Azure Arc resource bridge to onboard to Azure Arc-enabled SCVMM.

## CreateConfigKvaCustomerError

### PSSessionAccessDenied

[Learn about the cause and recommended action](https://go.microsoft.com/fwlink/?linkid=2298170).

### PSSessionGet-SCVMMServer

[Learn about the cause and recommended action](https://go.microsoft.com/fwlink/?linkid=2297694).

### PSSessionMIResultFailed

[Learn about the cause and recommended action](https://go.microsoft.com/fwlink/?linkid=2298171).

## KVAInvalidEntityCustomerError

### ValidateInsufficientLibSharePermission

[Learn about the cause and recommended action](https://go.microsoft.com/fwlink/?linkid=2297975).

### ValidateInsufficientPrivilege

[Learn about the cause and recommended action](https://go.microsoft.com/fwlink/?linkid=2298222).

### ValidateVlanIDNotAvailableOnVMNetwork

[Learn about the cause and recommended action](https://go.microsoft.com/fwlink/?linkid=2297976).

## PostOperationsError

### PostOperationTimeout

[Learn about the cause and recommended action](https://go.microsoft.com/fwlink/?linkid=2300587).

### PostOperationsErrorKubeadmControlPlane

[Learn about the cause and recommended action](https://go.microsoft.com/fwlink/?linkid=2301242).

## UpgradeError

### Upgrade_PreflightCheckErrorOnPrem

[Learn about the cause and recommended action](https://go.microsoft.com/fwlink/?linkid=2304710).

## Next steps

- [Troubleshoot common deployment errors](/azure/azure-arc/resource-bridge/troubleshoot-resource-bridge).
- [Support matrix for Azure Arc-enabled System Center Virtual Machine Manager](support-matrix-for-system-center-virtual-machine-manager.md).
