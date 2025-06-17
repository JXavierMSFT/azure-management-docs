---
title: Third-party monitoring with Prometheus and Grafana
description: Learn how to monitor your Azure Container Storage enabled by Azure Arc deployment using third-party monitoring with Prometheus and Grafana.
author: asergaz
ms.author: sergaz
ms.topic: how-to
ms.date: 08/26/2024

# Customer intent: "As a system administrator, I want to configure Prometheus and Grafana to monitor Azure Container Storage enabled by Azure Arc, so that I can effectively visualize and manage my containerized applications' performance and metrics."
---

# Third-party monitoring with Prometheus and Grafana

This article describes how to monitor your deployment using third-party monitoring with Prometheus and Grafana.

## Metrics

### Configure an existing Prometheus instance for use with Azure Container Storage enabled by Azure Arc

This guidance assumes that you previously worked with and/or configured Prometheus for Kubernetes. If you haven't previously done so, [see this overview](/azure/azure-monitor/containers/kubernetes-monitoring-enable#enable-prometheus-and-grafana) for more information about how to enable Prometheus and Grafana.

[See the metrics configuration section](azure-monitor-kubernetes.md#metrics-configuration) for information about the required Prometheus scrape configuration. Once you configure Prometheus metrics, you can deploy [Grafana](/azure/azure-monitor/visualize/grafana-plugin) to monitor and visualize your Azure services and applications.

## Logs

The Azure Container Storage enabled by Azure Arc logs are accessible through the Azure Kubernetes Service [kubelet logs](/azure/aks/kubelet-logs). You can also collect this log data using the [syslog collection feature in Azure Monitor Container Insights](/azure/azure-monitor/containers/container-insights-syslog).

## Next steps

[Azure Container Storage enabled by Azure Arc overview](overview.md)
