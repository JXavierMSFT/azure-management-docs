---
title: Onboard VMs to Azure Arc through the multicloud connector
description: Learn how to enable the Arc onboarding solution with the multicloud connector enabled by Azure Arc.
ms.topic: how-to
ms.date: 01/08/2025
---

# Onboard VMs to Azure Arc through the multicloud connector

The **Arc onboarding** solution of the multicloud connector autodiscovers VMs in a [connected public cloud](connect-to-aws.md), then installs the [Azure Connected Machine agent](/azure/azure-arc/servers/agent-overview) to onboard the VMs to Azure Arc. Currently, EC2 instances in AWS public cloud environments are supported.

This simplified experience lets you use Azure management services, such as Azure Monitor, providing a centralized way to manage Azure and AWS VMs together.

You can enable the **Arc onboarding** solution when you [connect your public cloud to Azure](connect-to-aws.md).

## Prerequisites

In addition to the [general prerequisites](connect-to-aws.md#prerequisites) for connecting a public cloud, be sure to meet the requirements for the **Arc onboarding** solution. This includes requirements for each EC2 instance to be onboarded to Azure Arc.

- You must have **AmazonEC2FullAccess** permissions in your public cloud.
- EC2 instances must meet the [general prerequisites for installing the Connected Machine agent](../servers/prerequisites.md).
- EC2 instances must have the SSM agent installed. Most EC2 instances have this preconfigured if you use a [supported OS](https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-instance-permissions.html).
- The **ArcForServerSSMRole** IAM role [attached on each EC2 instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html#attach-iam-role). This role attachment must be done after you upload your Cloud Formation Template in the Connector creation steps.

## AWS resource representation in Azure

After you connect your AWS cloud and enable the **Arc onboarding** solution, the Multicloud Connector creates a new resource group with the naming convention `aws_yourAwsAccountId`.

When EC2 instances are connected to Azure Arc, representations of these machines appear in this resource group. These resources are placed in Azure regions, using a [standard mapping scheme](resource-representation.md#region-mapping). You can filter for which Azure regions you would like to scan for. By default, all regions are scanned, but you can choose to exclude certain regions when you [configure the solution](connect-to-aws.md#add-your-public-cloud-in-the-azure-portal).

The `aws_yourAwsAccountId` resource group inherits permissions from its subscription. You can grant additional access to user accounts in your tenant as needed to enable specific scenarios.  

## Connectivity method

When creating the [**Arc onboarding** solution](connect-to-aws.md#add-your-public-cloud-in-the-azure-portal), you select whether the Connected Machine agent should connect to the internet via a public endpoint or by proxy server. If you select **Proxy server**, you must provide a **Proxy server URL** to which the EC2 instance can connect.

For more information, see [Connected machine agent network requirements](../servers/network-requirements.md?tabs=azure-cloud).

## Periodic sync options

The periodic sync time that you select when configuring the **Arc onboarding** solution determines how often your AWS account is scanned and synced to Azure. By enabling periodic sync, whenever a new EC2 instance that meets the prerequisites is discovered, the Arc agent is automatically installed. The periodic sync option will also help clean up your resources in Azure. For instance, if the EC2 instance is removed from AWS, the Arc server in Azure will also be deleted. This is only applicable to Arc servers created in the aws_accountId resource group.

If you prefer, you can turn periodic sync off when configuring this solution. If you do so, new EC2 instances aren't automatically onboarded to Azure Arc, because Azure doesn't scan for new instances.

## EC2 Filter Options

You can choose to filter to scan for EC2 based on AWS regions or AWS tags. You can select which regions you would like to scan for EC2 resources. You can also filter by AWS tag to only onboard EC2 machines that have the matching tag (case-insensitive) to be eligible for EC2 onboarding.

## Next steps

- Learn more about [managing connected servers through Azure Arc](../servers/overview.md).
- Learn about the [Multicloud Connector **Inventory** solution](view-multicloud-inventory.md).
