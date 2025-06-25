---
title: Known Issues for Workload Orchestration
description: This article provides a list of known issues for workload orchestration in Azure Arc, including workarounds.
author: SoniaLopezBravo
ms.author: sonialopez
ms.topic: troubleshooting-known-issue
ms.date: 06/08/2025
---

# Known issues for workload orchestration

This article provides a list of known issues for workload orchestration in Azure Arc. It includes workarounds and solutions for each issue. Known issues are updated as new issues are discovered and resolved.

## Custom location creation error when running onboarding script

When running the [infrastructure onboarding script](onboarding-scripts.md), you may encounter the following error during custom location creation:

```
'CredentialAdaptor' object has no attribute 'signed_session'
```

This issue affects Azure CLI version 2.70.0 and occurs when using the `az customlocation create` command. To resolve this issue, create the custom location via Azure portal

1. Navigate to the [Azure portal](https://portal.azure.com)
1. Click on **+ Create a resource** and search for "custom location".
1. In the **Basics** tab:
   - Select your subscription and resource group
   - Enter a name for your custom location
   - Select your Arc-enabled cluster
   - Select the appropriate extension (either `microsoft.testsymphonyex` or `microsoft.workloadorchestreation`)
   - Specify your namespace (the same value you use in the script)
1. Complete the creation process. 

After creating the custom location through the portal, run the onboarding script with `-skipCustomLocationCreation` set to `$true` to skip this step.


## Bulk deployment response lists a single deployed target 

When executing a [bulk deployment](bulk-deployment.md), the CLI response only lists one target in the `DeployedTargets` array, even though all the targets are successfully deployed. 

In the case of deployment failure, the CLI returns an error message and only lists one target under both `DeployedTargets` and `FailedTargets` arrays, even though the deployment fails. 

The message in the CLI response is a known issue and doesn't reflect the actual deployment status of the targets. Check the [workload orchestration portal](deploy.md) to verify the application status for each target. 

## Contact support

[!INCLUDE [form-feedback-note](includes/form-feeback.md)]

## Related content

- [Troubleshooting workload orchestration](troubleshooting.md)
- [Release notes for workload orchestration](release-notes.md)


 
