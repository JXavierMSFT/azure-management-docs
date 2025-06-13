---
title: SSH access to Azure Arc-enabled servers
description: Use SSH remoting to access and manage Azure Arc-enabled servers.
ms.date: 06/13/2025
ms.topic: concept-article
ms.custom: references_regions
---

# SSH access to Azure Arc-enabled servers

SSH for Arc-enabled servers enables SSH based connections to Arc-enabled servers without requiring a public IP address or additional open ports. This functionality can be used interactively, automated, or with existing SSH based tooling, expanding the impact of existing management tools on Azure Arc-enabled servers.

## Key benefits

SSH access to Arc-enabled servers provides the following benefits:

- No public IP address or open SSH ports required
- Access to Windows and Linux machines
- Ability to log in as a local user or an [Azure user (Linux only)](/azure/active-directory/devices/howto-vm-sign-in-azure-ad-linux)
- Support for other OpenSSH based tooling with config file support

## Prerequisites

- User Permissions: Owner or Contributor role assigned for the target Arc-enabled server.
- Arc-enabled server:
  - Hybrid Agent version: 1.31.xxxx or higher
  - SSH service ("sshd") must be enabled.

For Linux, install `openssh-server` via a package manager. You can check whether sshd is running on Linux by running the following command:

```shell
ps -aux | grep sshd
```

For Windows, see [Enable OpenSSH](/windows-server/administration/openssh/openssh_install_firstuse). You can check whether ssh is installed and running with the following commands:

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'

# Check the sshd service is running
Get-Service sshd
```

## Microsoft Entra authentication

If you use Microsoft Entra for authentication, you must install `aadsshlogin` and `aadsshlogin-selinux` (as appropriate) on the Arc-enabled server. These packages are installed with the `AADSSHLoginForLinux` VM extension.

You must also configure role assignments for the virtual machine (VM). Two Azure roles are used to authorize VM login:

- **Virtual Machine Administrator Login**: Users who have this role assigned can log in to an Azure VM with administrator privileges.
- **Virtual Machine User Login**: Users who have this role assigned can log in to an Azure VM with regular user privileges.

An Azure user with the Owner or Contributor role assigned for a VM doesn't automatically have privileges for Microsoft Entra login to the VM over SSH. There's an intentional (and audited) separation between the set of people who control virtual machines and the set of people who can access virtual machines.

> [!NOTE]
> The Virtual Machine Administrator Login and Virtual Machine User Login roles use `dataActions` and can be assigned at the management group, subscription, resource group, or resource scope. We recommend that you assign the roles at the management group, subscription, or resource level and not at the individual VM level. This practice avoids the risk of reaching the [Azure role assignments limit](/azure/role-based-access-control/troubleshoot-limits) per subscription.

### Availability

SSH access to Arc-enabled servers is currently supported in all cloud regions supported by Arc-enabled servers.

## Enable SSH access to Arc-enabled servers

To enable SSH access to Arc-enabled servers, follow the steps in this section.

### Register the HybridConnectivity resource provider

> [!NOTE]
> This is a one-time operation that needs to be performed on each subscription.

Check if the HybridConnectivity resource provider has been registered:

```azurecli
az provider show -n Microsoft.HybridConnectivity -o tsv --query registrationState
```

If the resource provider hasn't been registered, run the following command to register it:

```azurecli
az provider register -n Microsoft.HybridConnectivity
```

This operation can take 2-5 minutes to complete. Be sure the registration is complete before proceeding to the next step.

### Create default connectivity endpoint

This step must be completed for each Arc-enabled server. However, you may not need to run these commands to do so, as it should complete automatically at first connection.

#### [Azure CLI:](#tab/azure-cli)

```bash
az rest --method put --uri https://management.azure.com/subscriptions/<subscription>/resourceGroups/<resourcegroup>/providers/Microsoft.HybridCompute/machines/<arc enabled server name>/providers/Microsoft.HybridConnectivity/endpoints/default?api-version=2023-03-15 --body '{"properties": {"type": "default"}}'
```

> [!NOTE]
> If using Azure CLI from PowerShell, the following should be used.

```powershell
az rest --method put --uri https://management.azure.com/subscriptions/<subscription>/resourceGroups/<resourcegroup>/providers/Microsoft.HybridCompute/machines/<arc enabled server name>/providers/Microsoft.HybridConnectivity/endpoints/default?api-version=2023-03-15 --body '{\"properties\":{\"type\":\"default\"}}'
```

Validate endpoint creation:

 ```bash
az rest --method get --uri https://management.azure.com/subscriptions/<subscription>/resourceGroups/<resourcegroup>/providers/Microsoft.HybridCompute/machines/<arc enabled server name>/providers/Microsoft.HybridConnectivity/endpoints/default?api-version=2023-03-15
 ```

#### [Azure PowerShell](#tab/azure-powershell)

 ```powershell
Invoke-AzRestMethod -Method put -Path /subscriptions/<subscription>/resourceGroups/<resourcegroup>/providers/Microsoft.HybridCompute/machines/<arc enabled server name>/providers/Microsoft.HybridConnectivity/endpoints/default?api-version=2023-03-15 -Payload '{"properties": {"type": "default"}}'
```

Validate endpoint creation:

 ```powershell
 Invoke-AzRestMethod -Method get -Path /subscriptions/<subscription>/resourceGroups/<resourcegroup>/providers/Microsoft.HybridCompute/machines/<arc enabled server name>/providers/Microsoft.HybridConnectivity/endpoints/default?api-version=2023-03-15
 ```

 ---

### Install local command line tool

SSH functionality is provided in an Azure CLI extension and an Azure PowerShell module. Install the appropriate tool for your environment.

#### [Azure CLI](#tab/azure-cli)

```bash
az extension add --name ssh
```

#### [Azure PowerShell](#tab/azure-powershell)

```powershell
Install-Module -Name Az.Ssh -Scope CurrentUser -Repository PSGallery
Install-Module -Name Az.Ssh.ArcProxy -Scope CurrentUser -Repository PSGallery
```

---

### Enable functionality on your Arc-enabled server

In order to use the SSH connect feature, you must update the Service Configuration in the Connectivity Endpoint on the Arc-enabled server to allow SSH connection to a specific port. You may only allow connection to a single port. The CLI tools attempt to update the allowed port at runtime, but the port can be manually configured with the following command. If you're using a nondefault port for your SSH connection, replace port 22 with your desired port.

#### [Azure CLI](#tab/azure-cli)

```bash
az rest --method put --uri https://management.azure.com/subscriptions/<subscription>/resourceGroups/<resourcegroup>/providers/Microsoft.HybridCompute/machines/<arc enabled server name>/providers/Microsoft.HybridConnectivity/endpoints/default/serviceconfigurations/SSH?api-version=2023-03-15 --body "{\"properties\": {\"serviceName\": \"SSH\", \"port\": 22}}"
```

#### [Azure PowerShell](#tab/azure-powershell)

```powershell
Invoke-AzRestMethod -Method put -Path /subscriptions/<subscription>/resourceGroups/<resourcegroup>/providers/Microsoft.HybridCompute/machines/<arc enabled server name>/providers/Microsoft.HybridConnectivity/endpoints/default/serviceconfigurations/SSH?api-version=2023-03-15 -Payload '{"properties": {"serviceName": "SSH", "port": 22}}'
```

---

> [!NOTE]
> There may be a delay after updating the Service Configuration before you're able to connect.

### Optional: Install Microsoft Entra login extension

To use Microsoft Entra for authentication on your Linux machines, you must install `aadsshlogin` and `aadsshlogin-selinux` (as appropriate) on the Arc-enabled server. These packages are installed with the `AADSSHLoginForLinux` VM extension.

To add this extension in the Azure portal, navigate to your cluster, then in the service menu, under **Settings**, select **Extensions.** Select **Add**, then select **Azure AD based SSH Login – Azure Arc** and complete the installation. You can also install the extension locally via a package manager by running `apt-get install aadsshlogin` or the following command:

```bash
az connectedmachine extension create --machine-name <arc enabled server name> --resource-group <resourcegroup> --publisher Microsoft.Azure.ActiveDirectory --name AADSSHLogin --type AADSSHLoginForLinux --location <location>
```

## Examples

To view examples and more details, view the Az CLI documentation page for [az ssh](/cli/azure/ssh) or the Azure PowerShell documentation page for [Az.Ssh](/powershell/module/az.ssh).

## Disable SSH to Arc-enabled servers

If you need to remove SSH access to your Arc-enabled servers, follow the steps below.

 #### [Azure CLI](#tab/azure-cli)

1. Remove the SSH port and functionality from the Arc-enabled server:

    ```azurecli
    az rest --method delete --uri https://management.azure.com/subscriptions/<subscription>/resourceGroups/<resourcegroup>/providers/Microsoft.HybridCompute/machines/<arc enabled server name>/providers/Microsoft.HybridConnectivity/endpoints/default/serviceconfigurations/SSH?api-version=2023-03-15 --body '{\"properties\": {\"serviceName\": \"SSH\", \"port\": \"22\"}}'
    ```

1. Delete the default connectivity endpoint:

    ```azurecli
    az rest --method delete --uri https://management.azure.com/subscriptions/<subscription>/resourceGroups/<resourcegroup>/providers/Microsoft.HybridCompute/machines/<arc enabled server name>/providers/Microsoft.HybridConnectivity/endpoints/default?api-version=2023-03-15
    ```

#### [Azure PowerShell](#tab/azure-powershell)

1. Remove the SSH port and functionality from the Arc-enabled server:

   ```azurepowershell
   Invoke-AzRestMethod -Method delete -Path /subscriptions/<subscription>/resourceGroups/<resourcegroup>/providers/Microsoft.HybridCompute/machines/<arc enabled server name>/providers/Microsoft.HybridConnectivity/endpoints/default/serviceconfigurations/SSH?api-version=2023-03-15 -Payload '{"properties": {"serviceName": "SSH", "port": "22"}}'
   ```

1. Delete the default connectivity endpoint: 
   ```azurepowershell
   Invoke-AzRestMethod -Method delete -Path /subscriptions/<subscription>/resourceGroups/<resourcegroup>/providers/Microsoft.HybridCompute/machines/<arc enabled server name>/providers/Microsoft.HybridConnectivity/endpoints/default?api-version=2023-03-15
   ```

---

## Next steps

- Learn about [OpenSSH for Windows](/windows-server/administration/openssh/openssh_overview)
- Learn about troubleshooting [SSH access to Azure Arc-enabled servers](ssh-arc-troubleshoot.md).
- Learn about troubleshooting [agent connection issues](troubleshoot-agent-onboard.md).
