---
title: Azure Arc resource bridge overview
description: Learn how to use Azure Arc resource bridge to support VM self-servicing on Azure Local, VMware, and System Center Virtual Machine Manager.
ms.date: 04/22/2025
ms.topic: overview
ms.custom:
  - references_regions
  - build-2025
---

# What is Azure Arc resource bridge?

Azure Arc resource bridge is a prepackaged virtual appliance that runs as a Kubernetes-based management cluster deployed on your on-premises infrastructure (private cloud). It acts as a core component of the Azure Arc private cloud products and enables Azure-based management of on-premises resources. Azure Arc resource bridge provides a secure conduit between Azure and your on-premises infrastructure. It allows projection of on-premises resources into Azure as native ARM resources, enabling consistent governance, automation, and management with Azure tools. The resource bridge facilitates self-service provisioning and lifecycle management of Windows and Linux virtual machines directly from Azure. 

Azure Arc resource bridge integrates with the following private cloud platforms:

- Azure Local (via [Azure Arc VM management](/azure/azure-local/manage/azure-arc-vm-management-overview))

- VMware (via [Azure Arc-enabled VMware vSphere](../vmware-vsphere/overview.md))

- System Center Virtual Machine Manager (via [Azure Arc-enabled SCVMM](../system-center-virtual-machine-manager/overview.md))

Once deployed in your private cloud, the resource bridge is granted credentials to the local virtualization infrastructure, allowing it to project on-premises resources into Azure as Arc-enabled Azure Resource Manager (ARM) resources. This projection enables consistent management and automation using Azure tools, such as Azure Policy, and Azure CLI.

Arc resource bridge enables the following hybrid management capabilities:

* **Native Azure experience**: Projects on-premises VMs as Azure Resource Manager (ARM) resources, allowing you to view on-premises VMs in Azure and apply tags, policies, and extensions just like native Azure VMs.

* **VM self-service from Azure**: Create, manage, and delete VMs on-premises through the Azure portal or CLI.

* **Native Azure integration**: Extend Azure governance, monitoring, and automation capabilities to on-premises VMs. 

## Overview

Azure Arc Resource Bridge is a key component that allows Azure to manage on-premises private cloud infrastructure like VMware, SCVMM, or Azure Local environments. It acts as a local appliance VM that connects your on-premises infrastructure to Azure, enabling Azure to project and manage those resources as if they were native cloud assets.

To deliver this functionality, the resource bridge hosts additional Azure Arc components, including:

- **Custom location** – These define the target infrastructure for deployments. The custom location maps to your private cloud infrastructure. When you create a VM from Azure, you choose a custom location—Azure knows where to route the request and what infrastructure it maps to. For example, for Arc-enabled VMware, the custom location maps to an instance of vCenter. For Azure Arc VM management on Azure Local, it maps to an Azure Local instance. For more information, refer to [custom locations](..%5Cplatform%5Cconceptual-custom-locations.md).

- **Cluster extension** – A cluster extension enables private cloud capabilities on the resource bridge. The supported private cloud extensions are VMware, SCVMM and Azure Local.

- **Azure Arc agents** – Power the communication and control layer between Azure and your infrastructure.

:::image type="content" source="media/overview/architecture-overview.png" alt-text="Azure Arc resource bridge architecture diagram." border="false" lightbox="media/overview/architecture-overview.png":::

[!INCLUDE [arc-jumpstart-diagram](~/reusable-content/ce-skilling/azure/includes/arc-jumpstart-diagram.md)]

Custom locations and cluster extensions are both Azure resources linked to the Azure Arc resource bridge resource in Azure Resource Manager. When you create a VM from Azure, you select the custom location. Azure uses the custom location to determine the mapping to your private cloud infrastructure and routes the create VM request to your private cloud. A VM is created in your private cloud and a corresponding Azure resource is created in Azure as a representation of your on-premises VM in Azure. Azure Arc resource bridge enables this hybrid management of your on-premises resources from Azure.

Some VM creation inputs vary based on the private cloud:

- For VMware, you must specify the resource pool, network, and VM template.

- For Azure Local, you provide the custom location, network, and template.

The custom location, infrastructure and VM resources in Azure are *projections* of your on-premises environment. If an underlying on-premises resource becomes unhealthy, this status may be reflected in its corresponding Azure resource. 

Azure Arc resource bridge enables this projection and acts as the control plane that enables Azure to manage your private cloud infrastructure. If the resource bridge becomes unavailable or unhealthy, Azure may lose visibility or management capabilities of your on-premises resources. However, your on-premises resources, such as VMs running in vCenter, Azure Local or SCVMM, should not be affected and should continue to remain operational. 

## Benefits of Azure Arc resource bridge

Azure Arc resource bridge enables you to manage on-premises Windows and Linux virtual machines directly from Azure. Depending on the supported private cloud, you may be able to perform the following VM management operations:

- Provision and delete VMs from Azure portal or CLI

Start, stop, and restart VMs

- Control access using Azure RBAC and apply Azure tags

- Add, remove or update networking, disks, and VM size (CPU cores and memory)

- Enable guest management

- Install supported Azure Arc VM extensions (Ex: Azure Monitor, Azure Policy)

## Example scenarios

These examples show how Azure Arc resource bridge enables hybrid management of on-premises environments from Azure.

### Apply Azure Policy and other Azure services to on-premises VMware VMs

A customer deploys Arc Resource Bridge onto their on-premises VMware vSphere environment. They sign into the Azure portal, select the VMware VMs that they'd like to connect to Azure and select the option to enable Azure Arc. Once the VMs are Arc-enabled, they can manage these on-premises VMware VMs in Azure Resource Manager (ARM) as Arc-enabled machines. They can enable Azure services, such as Defender for Cloud and Azure Policy to extend security and governance from Azure to their on-premises VMware workloads.

:::image type="content" source="../media/overview/resource-bridge-vmware.png" alt-text="Diagram showing VMware VMs connected to Azure through Arc resource bridge." lightbox="../media/overview/resource-bridge-vmware.png":::

### Create physical Azure Local VMs on-premises from Azure

A customer has multiple datacenter locations in Canada and New York. They deploy Arc resource bridge in each datacenter and enable Azure Arc VM management of their Azure Local VMs in Azure. They can then sign into Azure Portal and see all their Arc-enabled VMs from the two physical locations in one central view from Azure Portal. From the Azure portal, they can:

View and manage all Arc-enabled VMs across both datacenters

- Centrally create new VMs in either datacenter from Azure Portal

Each new VM is provisioned on-premises in the selected location but appears in Azure Portal as an Arc-enabled VM. This setup enables centralized VM management across multiple physical sites all from within Azure.

:::image type="content" source="../media/overview/resource-bridge-multi-datacenter.png" alt-text="Diagram showing Azure Local VMs in two datacenters connected to Azure through Arc resource bridge." lightbox="../media/overview/resource-bridge-multi-datacenter.png":::

## Version and region support

### Supported regions

In order to use Arc resource bridge in a region, Arc resource bridge and the Arc-enabled private cloud must be supported in the region. For example, to use Arc resource bridge with Azure Local in East US, Arc resource bridge and the Arc VM management feature for Azure Local must be supported in East US. To confirm feature availability across regions for each private cloud provider, review their deployment guide and other documentation. There could be instances where Arc resource bridge is available in a region where the private cloud feature isn't yet available.

Arc resource bridge supports the following Azure regions:

* East US
* East US 2
* West US 2
* West US 3
* Central US
* North Central US
* South Central US
* US Gov Virginia

* Canada Central
* Australia East
* Australia SouthEast
* West Europe
* North Europe
* UK South
* UK West
* Sweden Central
* Italy North

* Japan East
* Southeast Asia
* East Asia
* Central India

### Regional resiliency

While Azure has redundancy features at every level of failure, if a service impacting event occurs, Azure Arc resource bridge currently doesn't support cross-region failover or other resiliency capabilities. If the service becomes unavailable, on-premises VMs continue to operate unaffected. Management from Azure is unavailable during that service outage.

### Private cloud environments

The following private cloud environments and their versions are officially supported for Arc resource bridge:

* VMware vSphere version 7.0, 8.0
* Azure Local
* SCVMM

### Supported versions

We generally recommend keeping your Arc resource bridge on a version released within the last 6 months or within the latest n-3 versions, whichever is more recent. This ensures your appliance benefits from the latest features, security updates, and refreshed internal components such as certificates. While the support policy includes the latest version and the three preceding versions (n-3), you must still upgrade at least once every 6 months, even if your current version is technically within the supported range. This is critical to maintain system health and compatibility. To estimate your last upgrade date, check your appliance version and its corresponding release date. For version release news, please refer to [Arc resource bridge release notes](release-notes.md)

### Private link support

Arc resource bridge currently doesn't support private link.

## Next steps

* Learn how [Azure Arc-enabled VMware vSphere extends Azure's governance and management capabilities to VMware vSphere infrastructure](../vmware-vsphere/overview.md).
* Learn how [Azure Arc-enabled SCVMM extends Azure's governance and management capabilities to System Center managed infrastructure](../system-center-virtual-machine-manager/overview.md).
* Learn about [provisioning and managing on-premises Windows and Linux VMs running on Azure Local instances](/azure/azure-local/manage/azure-arc-vm-management-overview).
* Review the [system requirements](system-requirements.md) for deploying and managing Arc resource bridge.
