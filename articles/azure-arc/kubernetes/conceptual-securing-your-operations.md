---
title: "Securing your operations in Azure Arc-enabled Kubernetes"
ms.date: 06/06/2025
ms.topic: concept-article
description: "Operational security guidance for managing Azure Arc-enabled Kubernetes clusters, including monitoring, logging, and incident response."
---

# Securing your operations

## Control who can use the Azure control plane to manage your cluster

It’s important to control access to the Azure control plane, including the subscription and resources that enable Azure Arc cloud management of your edge clusters using the Azure Portal, CLI, API, etc. You should review and implement the [Azure access control best practices](/azure/security/fundamentals/identity-management-best-practices), including multi-factor authentication, conditional access, and use Azure RBAC to control who can perform which operations with which clusters in just the same way you may already use for controlling access to your other Azure cloud resources.

Further, Microsoft-generated certificates are used to help secure the connection between your edge clusters and the Azure control plane. These certificates are stored as Kubernetes secrets, so it’s important that the Kubernetes secret store is itself encrypted. See section 4.2 for more advice on this.

## Control who can deploy to your cluster with Role Based Access Control (RBAC)

It’s also important to control access to the Kubernetes control plane (API server) itself, which is the means by which you can deploy and monitor your Kubernetes workloads.

For non-human access to the API server from workloads, use Kubernetes’ built-in RBAC to authorize only the specific service accounts that require it (see section 2.6 above for advice on issuing and protecting these service accounts).

For human access to the API server, Kubernetes doesn’t have built-in user accounts, so it’s recommended to integrate it with an external user account services such as Microsoft Entra ID. This is easy to do [if you’re running AKS enabled by Azure Arc on Azure Local](/azure/aks/hybrid/enable-authentication-microsoft-entra-id) or [if you’ve connected your own cluster](/azure/aks/hybrid/enable-authentication-microsoft-entra-id).

You can then create API server authorization policies that use these identities to control who can do what in which namespaces using RBAC, where it’s recommended to take a ‘least privilege’ approach of assigning each user or workload a role that has the minimum permissions required. You can set this up either using Kubernetes’ built-in RBAC or using Azure RBAC. Azure RBAC is recommended if you want to consistently manage and audit all your user authorization policies together in one central place, covering both your cloud and edge resources. Setting up these RBAC options is easy to do [if you’re running AKS enabled by Azure Arc on Azure Local](/azure/aks/hybrid/azure-rbac-23h2) or [if you’ve connected your own cluster](/azure/azure-arc/kubernetes/azure-rbac?tabs=kubernetes-latest). Your users can then use their Entra ID account to access the cluster (its API server) either directly or via an Azure proxy using the cluster connect capability.

More generally, follow standard best practice in separating development, test, and production clusters. And consider if production deployments to your clusters would be more reliably and securely managed by using a [GitOps approach](/azure/azure-arc/kubernetes/tutorial-use-gitops-flux2?tabs=azure-cli). If you enable this, then it’s also important to implement similar strong role-based access control for changes (pull requests) on the underlying source Git repository and branch used to define your deployments.

Finally, if you’re running AKS enabled by Azure Arc on Azure Local then you can also [download an admin client certificate for full admin access](/azure/aks/aksarc/retrieve-admin-kubeconfig). This should not typically be necessary, and it should only be used when required: e.g. to diagnose issues that can’t be troubleshooted any another way. This approach should also be used with care because it doesn’t use an Entra ID account and the per-user policies you may have set up above, and it the client certificate itself must be carefully stored and then deleted when no longer required.

### References
* [CIS Kubernetes Benchmark - Sections 1, 2, and 4](https://www.cisecurity.org/benchmark/kubernetes)
* [NIST Application Container Security Guide - Section 4.5.1-3](https://csrc.nist.gov/pubs/sp/800/190/final)
* [NSA Kubernetes Hardening Guidance – “Authentication and Authorization”](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF)
* [Kubernetes Security - OWASP Cheat Sheet Series – “Controlling access to the Kubernetes API”](https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html)

## Follow a secure container lifecycle as you deploy and run containers with Azure Policy for Kubernetes

Continue to follow the [Microsoft Containers Secure Supply Chain framework](/azure/security/container-secure-supply-chain/articles/container-secure-supply-chain-implementation/containers-secure-supply-chain-overview) through the deploy phase. (See above for the acquire, catalog, and build phases.)  This will help you deploy only from your own trusted registries, such as [Azure Container Registry](/azure/container-registry/). Use the registry’s access control mechanisms to ensure that only trusted clusters pull containers that may contain sensitive information. Azure Container Registry supports both [Role Based Access Control (RBAC)](/azure/container-registry/container-registry-rbac-built-in-roles-overview?tabs=registries-configured-with-rbac-registry-abac-repository-permissions) and [Attribute Based Access Control (ABAC)](/azure/container-registry/container-registry-rbac-abac-repository-permissions?tabs=azure-portal) to further scope assignments to specific repositories.

Additionally, enforce best practice standards for container security hygiene through Azure Policy’s extension for AKS and Arc-enabled clusters. For example, you can validate that all pods meet the Pod Security Standards in a low-code approach by using [Azure Policy's built-in definitions}(/azure/governance/policy/samples/built-in-policies#kubernetes). You can also deploy the [Azure Policy extension](/azure/governance/policy/concepts/policy-for-kubernetes?toc=%2Fazure%2Fazure-arc%2Fkubernetes%2Ftoc.json&bc=%2Fazure%2Fazure-arc%2Fkubernetes%2Fbreadcrumb%2Ftoc.json#install-azure-policy-extension-for-azure-arc-enabled-kubernetes), which extends [Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/), to your edge Kubernetes cluster to apply pod-based security enforcement at-scale. We recommend that you first apply policy assignments in ‘audit’ mode, which will provide an aggregated list of non-compliant results at a per-Kubernetes resource, per-policy granularity, allowing you to spot and remediate any existing issues with your running deployments first. Once you have fixed the non-compliant violations in your environment, you can then update the policy assignment to ‘deny’ mode, using Azure Policy’s rich safe-deployment mechanisms to rollout policy enforcement gradually across resources. By applying policies in enforcement mode, you will actively prevent any further deviations. 

### References
* [Section 4.2 of the NIST Application Container Security Guide](https://csrc.nist.gov/pubs/sp/800/190/final)
* [Kubernetes Security - OWASP Cheat Sheet Series – “Continuously assess the privileges used by containers” and “Use Pod Security Admission”](https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html)

## Detect emerging threats including monitoring control plane changes

Help ensure you have a way to detect threats as they arise in your clusters.

You can deploy the [Defender for Containers extension](/azure/defender-for-cloud/defender-for-containers-introduction#run-time-protection-for-kubernetes-nodes-and-clusters) to your Kubernetes cluster at the edge. This includes a [sensor](/azure/defender-for-cloud/defender-for-containers-enable?tabs=aks-deploy-portal%2Ck8s-deploy-cli%2Ck8s-verify-asc%2Ck8s-remove-arc%2Caks-removeprofile-api&pivots=defender-for-container-arc) that collects logs and sends them to Defender for Cloud by where they can be analyzed for anomalous behaviors that might indicate an attack or used for forensics after a possible incident. See the [support matrix](/azure/defender-for-cloud/support-matrix-defender-for-containers?tabs=azureva%2Carcrt%2Carcspm%2Carcnet) for which features are supported, as a preview or GA release, on which cluster types. In turn, Defender for Cloud can send events for analysis as part of [Microsoft Defender XDR](/azure/defender-for-cloud/concept-integration-365) incident detection and response solution.

If you’re running on AKS enabled by Azure Arc on Azure Local, you can also [configure it to send Kubernetes audit logs to Azure Monitor](/azure/aks/aksarc/kubernetes-monitor-audit-events) (Log Analytics Workspace). Note the related advice for monitoring your workloads themselves in section 2.8 above, as well [best practices}(/azure/azure-monitor/containers/best-practices-containers) for monitoring clusters that covers reliability, cost optimization, performance, as well as security.

Beyond this, look to build an incident response plan and practice using it. The details of such a plan depend greatly on your overall deployment environment, and the security operations tools you use: see this [guidance](/security/operations/incident-response-overview) for more. But at minimum, think about how you’d preserve your cluster’s state (retain audit logs, snapshot suspicious states) and how you’d recover it to a known-good state: see section 4.4 for more on that.

### References
* [CIS Kubernetes Benchmark - Section 3.2](https://www.cisecurity.org/benchmark/kubernetes)
* [NIST Application Container Security Guide - Section 4.4.4](https://csrc.nist.gov/pubs/sp/800/190/final)
* [NSA Kubernetes Hardening Guidance – “Threat detection”](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF)
* [Kubernetes Security - OWASP Cheat Sheet Series – “Logging”](https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html)

## Leverage deployment strategies to achieve zero-downtime updates

Critical security updates should not compromise the reliability and availability of your workloads, even when rolled out urgently. Choose a [Kubernetes deployment strategy](https://azure.microsoft.com/solutions/kubernetes-on-azure/deployment-strategy/) that best helps maintain high availability in your environment, and consider implementing [readiness- and liveness-probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/) allow Kubernetes to better learn about the state of your workloads as it maintains your deployment. Combined with gradual rollouts and traffic management policies at your ingress load-balancer, you can leverage Kubernetes to drive updates without interrupting the availability of your applications.

## Next steps

- Learn about [securing your data](conceptual-securing-your-data.md)
- Return to the top of this [security book](conceptual-security-book.md)