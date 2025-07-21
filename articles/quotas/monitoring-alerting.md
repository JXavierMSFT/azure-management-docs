---
title:  Quota monitoring & alerting
description: Learn about monitoring and alerting for quota usage.
ms.date: 11/29/2023
ms.topic: how-to
# Customer intent: "As a cloud resource manager, I want to monitor and set up alerts for quota usage, so that I can proactively manage resources and prevent service disruptions."
---

# Quota monitoring and alerting

Monitoring and alerting in Azure provides real-time insights into resource utilization, enabling proactive issue resolution and resource optimization. Use monitoring and alerting to help detect anomalies and potential issues before they impact services.

To view the features on **Quotas** page, sign in to the [Azure portal](https://portal.azure.com) and enter "quotas" into the search box, then select **Quotas**.

> [!NOTE]
> When monitoring and alerting is enabled for your account, the Quotas in **MyQuotas** will be highlighted and clickable.

## Monitoring

Monitoring for quotas lets you proactively manage your Azure resources. Azure sets predefined limits, or quotas, for various resources like **Compute**, **Azure Machine Learning**, and **HPC Cache**. This monitoring involves continuous tracking of resource usage to ensure it remains within allocated limits, including notifications when these limits are approached or reached.

## Alerting

Quota alerts in Azure are notifications triggered when the usage of a specific Azure resource nears the **predefined quota limit**. These alerts are crucial for informing Azure users and administrators about resource consumption, facilitating proactive resource management. Azure’s alert rule capabilities allow you to create multiple alert rules for a given quota or across quotas in your subscription.

For more information, see [Create alerts for quotas](how-to-guide-monitoring-alerting.md).

> [!NOTE]
> [General Role based access control](/azure/azure-monitor/alerts/alerts-overview#azure-role-based-access-control-for-alerts) applies while creating alerts.  

## Next steps

- Learn [how to create quota alerts](how-to-guide-monitoring-alerting.md).
- Learn more about [alerts](/azure/azure-monitor/alerts/alerts-overview)
- Learn about [Azure Resource Graph](/azure/governance/resource-graph/overview)

