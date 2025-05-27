---
title: Use the Azure Key Vault Secret Store extension to sync secrets to the Kubernetes secret store for offline access in Azure Arc-enabled Kubernetes clusters
description: The Azure Key Vault Secret Store extension for Kubernetes ("SSE") automatically synchronizes secrets from an Azure Key Vault to a Kubernetes cluster for offline access.
ms.date: 09/26/2024
ms.topic: how-to
ms.custom: references_regions, ignite-2024
# Customer intent: "As a Kubernetes administrator, I want to automatically synchronize secrets from Azure Key Vault to my Kubernetes cluster for offline access, so that I can manage critical business assets securely, even in semi-disconnected environments."
---

# Use the Secret Store extension to fetch secrets for offline access in Azure Arc-enabled Kubernetes clusters

The Azure Key Vault Secret Store extension for Kubernetes ("SSE") automatically synchronizes secrets from an [Azure Key Vault](/azure/key-vault/general/overview) to an [Azure Arc-enabled Kubernetes cluster](overview.md) for offline access. This means you can use Azure Key Vault to store, maintain, and rotate your secrets, even when running your Kubernetes cluster in a semi-disconnected state. Synchronized secrets are stored in the cluster [secret store](https://Kubernetes.io/docs/concepts/configuration/secret/), making them available as Kubernetes secrets to be used in all the usual ways: mounted as data volumes, or exposed as environment variables to a container in a pod.

Synchronized secrets are critical business assets, so the SSE secures them through isolated namespaces and nodes, role-based access control (RBAC) policies, and limited permissions for the secrets synchronizer. For extra protection, [encrypt](https://Kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) the Kubernetes secret store on your cluster.

> [!TIP]
> The SSE is recommended for scenarios where offline access is necessary, or if you need secrets synced into the Kubernetes secret store. If you don't need these features, you can use the [Azure Key Vault Secrets Provider extension](tutorial-akv-secrets-provider.md) for secret management in your Arc-enabled Kubernetes clusters. It's not recommended to run both the online Azure Key Vault Secrets Provider extension and the offline SSE side-by-side in a cluster.

This article shows you how to install and configure the SSE as an [Azure Arc-enabled Kubernetes extension](conceptual-extensions.md).

> [!IMPORTANT]
> SSE is currently in PREVIEW.
> See the [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/) for legal terms that apply to Azure features that are in beta, preview, or otherwise not yet released into general availability.

## Prerequisites

- An Arc-enabled cluster. This can be one that you [connected to yourself](quickstart-connect-cluster.md) (the examples throughout this guide use a [K3s](https://k3s.io/) cluster) or a Microsoft-managed [AKS enabled by Azure Arc](/azure/aks/hybrid/aks-overview) cluster. The cluster must be running Kubernetes version 1.27 or higher.
- Ensure you meet the [general prerequisites for cluster extensions](extensions.md#prerequisites), including the latest version of the `k8s-extension` Azure CLI extension.
- cert-manager is required to support TLS for intracluster log communication. The examples later in this guide direct you though installation. For more information about cert-manager, see [cert-manager.io](https://cert-manager.io/)

Install the [Azure CLI](/cli/azure/install-azure-cli-linux?pivots=apt) and sign in, if you haven't already:

```azurecli
az login
```

Before you begin, set environment variables to be used for configuring Azure and cluster resources. If you already have a managed identity, Azure Key Vault, or other resource listed here, update the names in the environment variables to reflect those resources. Note that the KEYVAULT_NAME must be globally unique; keyvault creation will fail later if this name is already in use within Azure.

```azurecli
export RESOURCE_GROUP="AzureArcTest"
export CLUSTER_NAME="AzureArcTest1"
export LOCATION="EastUS"
export SUBSCRIPTION="$(az account show --query id --output tsv)"
az account set --subscription "${SUBSCRIPTION}"
export AZURE_TENANT_ID="$(az account show -s $SUBSCRIPTION --query tenantId --output tsv)"
export CURRENT_USER="$(az ad signed-in-user show --query userPrincipalName --output tsv)"
export KEYVAULT_NAME="my-UNIQUE-kv-name"
export KEYVAULT_SECRET_NAME="my-secret"
export USER_ASSIGNED_IDENTITY_NAME="my-identity"
export FEDERATED_IDENTITY_CREDENTIAL_NAME="my-credential"
export KUBERNETES_NAMESPACE="my-namespace"
export SERVICE_ACCOUNT_NAME="my-service-account"
```

## Activate workload identity federation in your cluster

The SSE uses a feature called [workload identity federation](conceptual-workload-identity.md) to access and synchronize Azure Key Vault secrets. This section describes how to set the feature up. The following sections will explain how it's used in detail.

### [Arc-enabled Kubernetes](#tab/arc-k8s)

> [!TIP]
> The following steps are based on the [How-to guide](/azure/azure-arc/kubernetes/workload-identity) for configuring Arc-enabled Kubernetes with workload identity federation. Refer to that documentation for any additional assistance.

If your cluster isn't yet connected to Azure Arc, [follow these steps](quickstart-connect-cluster.md). During these steps, enable workload identity federation as part of the `connect` command:

```azurecli
az connectedk8s connect --name ${CLUSTER_NAME} --resource-group ${RESOURCE_GROUP} --enable-oidc-issuer
```

If your cluster is already connected to Azure Arc, enable workload identity using the `update` command:

```azurecli
az connectedk8s update --name ${CLUSTER_NAME} --resource-group ${RESOURCE_GROUP} --enable-oidc-issuer
```

Now configure your cluster to issue Service Account tokens with a new issuer URL (`service-account-issuer`) that enables Microsoft Entra ID to find the public keys necessary for it to validate these tokens. These public keys are for the cluster's own service account token issuer, and they were obtained and cloud-hosted at this URL as a result of the `--enable-oidc-issuer` option that you set earlier.

Optionally, you can also configure limits on the SSE's own permissions as a privileged resource running in the control plane by configuring [`OwnerReferencesPermissionEnforcement`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#ownerreferencespermissionenforcement) [admission controller](https://Kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#how-do-i-turn-on-an-admission-controller). This admission controller constrains how much the SSE can change other objects in the cluster.

1. Configure your [kube-apiserver](https://Kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/) with the issuer URL field and permissions enforcement. The following example is for a k3s cluster. Your cluster may have different means for changing API server arguments: `--kube-apiserver-arg="--service-account-issuer=${SERVICE_ACCOUNT_ISSUER}" and --kube-apiserver-arg="--enable-admission-plugins=OwnerReferencesPermissionEnforcement"`.

   - Get the service account issuer URL.

      ```console
      export SERVICE_ACCOUNT_ISSUER="$(az connectedk8s show --name ${CLUSTER_NAME} --resource-group ${RESOURCE_GROUP} --query "oidcIssuerProfile.issuerUrl" --output tsv)"
      echo $SERVICE_ACCOUNT_ISSUER
      ```

   - Open the K3s server configuration file.

      ```console
      sudo nano /etc/systemd/system/k3s.service
      ```

   - Edit the server configuration to look like the following example, replacing <SERVICE_ACCOUNT_ISSUER> with the previous output from `echo $SERVICE_ACCOUNT_ISSUER`, remembering to include the trailing forward slash of this URL: 
   
      ```console
      ExecStart=/usr/local/bin/k3s \
       server --write-kubeconfig-mode=644 \
          --kube-apiserver-arg="--service-account-issuer=<SERVICE_ACCOUNT_ISSUER>" \
          --kube-apiserver-arg="--enable-admission-plugins=OwnerReferencesPermissionEnforcement"
      ```

1. Restart your kube-apiserver.

    ```console
   sudo systemctl daemon-reload
   sudo systemctl restart k3s
   ```

### [AKS on Azure Local](#tab/aks-local)

Use the [How-to guide](/azure/aks/hybrid/workload-identity) to activate workload identity federation on AKS on Azure Local by using the `--enable-oidc-issuer` flag.

Return to these steps after the initial activation. There's no need to complete the remainder of that guide.

Validate the activation has been successful by obtaining the cluster's service account issuer URL. You'll use this URL in the following steps:  

   ```console
   export SERVICE_ACCOUNT_ISSUER="$(az connectedk8s show --name ${CLUSTER_NAME} --resource-group ${RESOURCE_GROUP} --query "oidcIssuerProfile.issuerUrl" --output tsv)"
   echo $SERVICE_ACCOUNT_ISSUER
   ```

### [AKS Edge Essentials](#tab/aks-ee)

Use the [How-to guide](/azure/aks/hybrid/aks-edge-workload-identity) to activate workload identity federation on AKS Edge Essentials. 

Return to these steps after the initial activation. There's no need to complete the remainder of that guide.

Validate the activation has been successful by obtaining the cluster's service account issuer URL. You'll use this URL in the following steps:  

   ```console
   export SERVICE_ACCOUNT_ISSUER="$(az connectedk8s show --name ${CLUSTER_NAME} --resource-group ${RESOURCE_GROUP} --query "oidcIssuerProfile.issuerUrl" --output tsv)"
   echo $SERVICE_ACCOUNT_ISSUER
   ```

---

## Create a secret and configure an identity to access it

To access and synchronize a given Azure Key Vault secret, the SSE requires access to an Azure managed identity with appropriate Azure permissions to access that secret. The managed identity must be linked to a Kubernetes service account using the workload identity feature that you activated earlier. The SSE uses the associated federated Azure managed identity to pull secrets from Azure Key Vault to your Kubernetes secret store. The following sections describe how to set this up.

### Create an Azure Key Vault

[Create an Azure Key Vault](/azure/key-vault/secrets/quick-create-cli) and add a secret. If you already have an Azure Key Vault and secret, you can skip this section.

1. Create an Azure Key Vault:

   ```azurecli
   az keyvault create --resource-group "${RESOURCE_GROUP}" --location "${LOCATION}" --name "${KEYVAULT_NAME}" --enable-rbac-authorization
   ```

1. Give yourself 'Secrets Officer' permissions on the vault, so you can create a secret:

   ```azurecli
   az role assignment create --role "Key Vault Secrets Officer" --assignee ${CURRENT_USER} --scope /subscriptions/${SUBSCRIPTION}/resourcegroups/${RESOURCE_GROUP}/providers/Microsoft.KeyVault/vaults/${KEYVAULT_NAME}
   ```

1. Create a secret and update it so you have two versions:

   ```azurecli
   az keyvault secret set --vault-name "${KEYVAULT_NAME}" --name "${KEYVAULT_SECRET_NAME}" --value 'Hello!'
   az keyvault secret set --vault-name "${KEYVAULT_NAME}" --name "${KEYVAULT_SECRET_NAME}" --value 'Hello2'
   ```

### Create a user-assigned managed identity

Next, create a user-assigned managed identity and give it permissions to access the Azure Key Vault. If you already have a managed identity with Key Vault Reader and Key Vault Secrets User permissions to the Azure Key Vault, you can skip this section. For more information, see [Create a user-assigned managed identities](/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities?pivots=identity-mi-methods-azp#create-a-user-assigned-managed-identity) and [Using Azure RBAC secret, key, and certificate permissions with Key Vault](/azure/key-vault/general/rbac-guide?tabs=azure-cli#using-azure-rbac-secret-key-and-certificate-permissions-with-key-vault).

1. Create the user-assigned managed identity:

   ```azurecli
   az identity create --name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}" --location "${LOCATION}" --subscription "${SUBSCRIPTION}"
   ```

1. Give the identity Key Vault Reader and Key Vault Secrets User permissions. You may need to wait a moment for replication of the identity creation before these commands succeed:

   ```azurecli
   export USER_ASSIGNED_CLIENT_ID="$(az identity show --resource-group "${RESOURCE_GROUP}" --name "${USER_ASSIGNED_IDENTITY_NAME}" --query 'clientId' -otsv)"
   az role assignment create --role "Key Vault Reader" --assignee "${USER_ASSIGNED_CLIENT_ID}" --scope /subscriptions/${SUBSCRIPTION}/resourcegroups/${RESOURCE_GROUP}/providers/Microsoft.KeyVault/vaults/${KEYVAULT_NAME}
   az role assignment create --role "Key Vault Secrets User" --assignee "${USER_ASSIGNED_CLIENT_ID}" --scope /subscriptions/${SUBSCRIPTION}/resourcegroups/${RESOURCE_GROUP}/providers/Microsoft.KeyVault/vaults/${KEYVAULT_NAME}
   ```

### Create a federated identity credential

Create a Kubernetes service account for the workload that needs access to secrets. Then, create a [federated identity credential](https://azure.github.io/azure-workload-identity/docs/topics/federated-identity-credential.html) to link between the managed identity, the OIDC service account issuer, and the Kubernetes Service Account.

1. Create a Kubernetes Service Account that will be federated to the managed identity. Annotate it with details of the associated user-assigned managed identity.

   ``` console
   kubectl create ns ${KUBERNETES_NAMESPACE}
   ```

   ``` console
   cat <<EOF | kubectl apply -f -
     apiVersion: v1
     kind: ServiceAccount
     metadata:
       name: ${SERVICE_ACCOUNT_NAME}
       namespace: ${KUBERNETES_NAMESPACE}
   EOF
   ```

1. Create a federated identity credential:

   ```azurecli
   az identity federated-credential create --name ${FEDERATED_IDENTITY_CREDENTIAL_NAME} --identity-name ${USER_ASSIGNED_IDENTITY_NAME} --resource-group ${RESOURCE_GROUP} --issuer ${SERVICE_ACCOUNT_ISSUER} --subject system:serviceaccount:${KUBERNETES_NAMESPACE}:${SERVICE_ACCOUNT_NAME}
   ```

## Install the SSE

The SSE is available as an Azure Arc extension. An [Azure Arc-enabled Kubernetes cluster](overview.md) can be extended with [Azure Arc-enabled Kubernetes extensions](extensions.md). Extensions enable Azure capabilities on your connected cluster and provide an Azure Resource Manager-driven experience for the extension installation and lifecycle management.

[cert-manager](https://cert-manager.io/) and [trust-manager](https://cert-manager.io/docs/trust/trust-manager/) are also required for secure communication of logs between cluster services and must be installed before the Arc extension.

1. Install cert-manager.
   ```azurecli
   helm repo add jetstack https://charts.jetstack.io/ --force-update
   helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.16.2 --set crds.enabled=true 
   ```

1. Install trust-manager.

   ```azurecli
   helm upgrade trust-manager jetstack/trust-manager --install --namespace cert-manager --wait
   ```

1. Install the SSE to your Arc-enabled cluster using the following command:

   ``` console
   az k8s-extension create \
     --cluster-name ${CLUSTER_NAME} \
     --cluster-type connectedClusters \
     --extension-type microsoft.azure.secretstore \
     --resource-group ${RESOURCE_GROUP} \
     --release-train preview \
     --name ssarcextension \
     --scope cluster 
   ```

   If desired, you can optionally modify the default rotation poll interval by adding `--configuration-settings rotationPollIntervalInSeconds=<time_in_seconds>`:

   | Parameter name                    | Description                         | Default value                         |
   |---------------------------------|-------------------------------------------------------------------------------|----------------------------------------------|
   | `rotationPollIntervalInSeconds`          | Specifies how quickly the SSE checks or updates the secret it's managing.       | `3600` (1 hour)                                             |

## Configure the SSE

Configure the installed extension with information about your Azure Key Vault and which secrets to synchronize to your cluster by defining instances of Kubernetes [custom resources](https://Kubernetes.io/docs/concepts/extend-Kubernetes/api-extension/custom-resources/). You create two types of custom resources:

- A `SecretProviderClass` object to define the connection to the Key Vault.
- A `SecretSync` object for each secret to be synchronized.

### Create a `SecretProviderClass` resource

The `SecretProviderClass` resource is used to define the connection to the Azure Key Vault, the identity to use to access the vault, which secrets to synchronize, and the number of versions of each secret to retain locally.

You need a separate `SecretProviderClass` for each Azure Key Vault you intend to synchronize, for each identity used for access to an Azure Key Vault, and for each target Kubernetes namespace.

Create one or more `SecretProviderClass` YAML files with the appropriate values for your Key Vault and secrets by following this example.

``` yaml
cat <<EOF > spc.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: secret-provider-class-name                      # Name of the class; must be unique per Kubernetes namespace
  namespace: ${KUBERNETES_NAMESPACE}                    # Kubernetes namespace to make the secrets accessible in
spec:
  provider: azure
  parameters:
    clientID: "${USER_ASSIGNED_CLIENT_ID}"               # Managed Identity Client ID for accessing the Azure Key Vault with.
    keyvaultName: ${KEYVAULT_NAME}                       # The name of the Azure Key Vault to synchronize secrets from.
    objects: |
      array:
        - |
          objectName: ${KEYVAULT_SECRET_NAME}            # The name of the secret to sychronize.
          objectType: secret
          objectVersionHistory: 2                       # [optional] The number of versions to synchronize, starting from latest.
    tenantID: "${AZURE_TENANT_ID}"                       # The tenant ID of the Key Vault 
EOF
```

### Create a `SecretSync` object

Each synchronized secret also requires a `SecretSync` object, to define cluster-specific information. Here you specify information such as the name of the secret in your cluster and names for each version of the secret stored in your cluster.

Create one `SecretSync` object YAML file for each secret, following this template. The Kubernetes namespace should match the namespace of the matching `SecretProviderClass`.

```yaml
cat <<EOF > ss.yaml
apiVersion: secret-sync.x-k8s.io/v1alpha1
kind: SecretSync
metadata:
  name: secret-sync-name                                  # Name of the object; must be unique per Kubernetes namespace
  namespace: ${KUBERNETES_NAMESPACE}                      # Kubernetes namespace
spec:
  serviceAccountName: ${SERVICE_ACCOUNT_NAME}             # The Kubernetes service account to be given permissions to access the secret.
  secretProviderClassName: secret-provider-class-name     # The name of the matching SecretProviderClass with the configuration to access the AKV storing this secret
  secretObject:
    type: Opaque
    data:
    - sourcePath: ${KEYVAULT_SECRET_NAME}/0                # Name of the secret in Azure Key Vault with an optional version number (defaults to latest)
      targetKey: ${KEYVAULT_SECRET_NAME}-data-key0         # Target name of the secret in the Kubernetes secret store (must be unique)
    - sourcePath: ${KEYVAULT_SECRET_NAME}/1                # [optional] Next version of the AKV secret. Note that versions of the secret must match the configured objectVersionHistory in the secrets provider class 
      targetKey: ${KEYVAULT_SECRET_NAME}-data-key1         # [optional] Next target name of the secret in the K8s secret store
EOF
```

### Apply the configuration CRs

Apply the configuration custom resources (CRs) using the `kubectl apply` command:

``` bash
kubectl apply -f ./spc.yaml
kubectl apply -f ./ss.yaml
```

The SSE automatically looks for the secrets and begins syncing them to the cluster.

### View configuration options

To view additional configuration options for these two custom resource types, use the `kubectl describe` command to inspect the CRDs in the cluster:

```bash
# Get the name of any applied CRD(s)
kubectl get crds -o custom-columns=NAME:.metadata.name

# View the full configuration options and field parameters for a given CRD
kubectl describe crd secretproviderclass
kubectl describe crd secretsync
```

## Observe secrets synchronizing to the cluster

Once the configuration is applied, secrets begin syncing to the cluster automatically at the cadence specified when installing the SSE.

### View synchronized secrets

View the secrets synchronized to the cluster by running the following command:

```bash
# View a list of all secrets in the namespace
kubectl get secrets -n ${KUBERNETES_NAMESPACE}

# View details of all secrets in the namespace
kubectl get secrets -n ${KUBERNETES_NAMESPACE} -o yaml
```

### View last sync status

To view the status of the most recent synchronization for a given secret, use the `kubectl describe` command for the `SecretSync` object. The output includes the secret creation timestamp, the versions of the secret, and detailed status messages for each synchronization event. This output can be used to diagnose connection or configuration errors, and to observe when the secret value changes.

```bash
kubectl describe secretsync secret-sync-name -n ${KUBERNETES_NAMESPACE}
```

### View secrets values

To view the synchronized secret values, now stored in the Kubernetes secret store, use the following command:

```bash
kubectl get secret secret-sync-name -n ${KUBERNETES_NAMESPACE} -o jsonpath="{.data.${KEYVAULT_SECRET_NAME}-data-key0}" | base64 -d
kubectl get secret secret-sync-name -n ${KUBERNETES_NAMESPACE} -o jsonpath="{.data.${KEYVAULT_SECRET_NAME}-data-key1}" | base64 -d
```

## Troubleshooting

The SSE is a Kubernetes deployment that contains a pod with two containers: the controller, which manages storing secrets in the cluster, and the provider, which manages access to, and pulling secrets from, the Azure Key Vault. Each synchronized secret has a `SecretSync` object that contains the status of the synchronization of that secret from Azure Key Vault to the cluster secret store.

To troubleshoot an issue, start by looking at the state of the `SecretSync` object, as described in [View last sync status](#view-last-sync-status). The following table lists common status types, their meanings, and potential troubleshooting steps to resolve errors.

| SecretSync Status Type     | Details      | Steps to fix/investigate further    |
|------------|--------------|-------------------------------------|
| `CreateSucceeded` | The secret was created successfully. | n/a |
| `CreateFailedProviderError` | Secret creation failed due to some issue with the provider (connection to Azure Key Vault). This failure could be due to internet connectivity, insufficient permissions for the identity syncing secrets, misconfiguration of the `SecretProviderClass`, or other issues. | Investigate further by looking at the logs of the provider using the following commands: <br>```kubectl get pods -n azure-secret-store``` <br>```kubectl logs <secret-sync-controller-pod-name> -n azure-secret-store --container='provider-azure-installer'``` |
| `CreateFailedInvalidLabel` | The secret creation failed because the secret already exists without the correct Kubernetes label that the SSE uses to manage its secrets.| Remove the existing label and secret and allow the SSE to recreate the secret: ```kubectl delete secret <secret-name>``` <br>To force the SSE to recreate the secret faster than the configured rotation poll interval, delete the `SecretSync` object (```kubectl delete secretsync <secret-name>```) and reapply the secret sync class (```kubectl apply -f <path_to_secret_sync>```). |
| `CreateFailedInvalidAnnotation` | Secret creation failed because the secret already exists without the correct Kubernetes annotation that the SSE uses to manage its secrets. | Remove the existing annotation and secret and allow the SSE to recreate the secret: ```kubectl delete secret <secret-name>``` <br>To force the SSE to recreate the secret faster than the configured rotation poll interval, delete the `SecretSync` object (```kubectl delete secretsync <secret-name>```) and reapply the secret sync class (```kubectl apply -f <path_to_secret_sync>```). |
| `UpdateNoValueChangeSucceeded` | The SSE checked Azure Key Vault for updates at the end of the configured poll interval, but there were no changes to sync. | n/a |
| `UpdateValueChangeOrForceUpdateSucceeded` | The SSE checked Azure Key Vault for updates and successfully updated the value. | n/a |
| `UpdateFailedInvalidLabel` | Secret update failed because the label on the secret that the SSE uses to manage its secrets was modified. | Remove the existing label and secret, and allow the SSE to recreate the secret: ```kubectl delete secret <secret-name>``` <br>To force the SSE to recreate the secret faster than the configured rotation poll interval, delete the `SecretSync` object (```kubectl delete secretsync <secret-name>```) and reapply the secret sync class (```kubectl apply -f <path_to_secret_sync>```). |
| `UpdateFailedInvalidAnnotation` | Secret update failed because the annotation on the secret that the SSE uses to manage its secrets was modified. | Remove the existing annotation and secret and allow the SSE to recreate the secret: ```kubectl delete secret <secret-name>``` <br>To force the SSE to recreate the secret faster than the configured rotation poll interval, delete the `SecretSync` object (```kubectl delete secretsync <secret-name>```) and reapply the secret sync class (```kubectl apply -f <path_to_secret_sync>```). |
| `UpdateFailedProviderError` | Secret update failed due to some issue with the provider (connection to Azure Key Vault). This failure could be due to internet connectivity, insufficient permissions for the identity syncing secrets, configuration of the `SecretProviderClass`, or other issues. | Investigate further by looking at the logs of the provider using the following commands: <br>```kubectl get pods -n azure-secret-store``` <br>```kubectl logs <secret-sync-controller-pod-name> -n azure-secret-store --container='provider-azure-installer'``` |
| `UserInputValidationFailed` | Secret update failed because the secret sync class was configured incorrectly (such as an invalid secret type). | Review the secret sync class definition and correct any errors. Then, delete the `SecretSync` object (```kubectl delete secretsync <secret-name>```), delete the secret sync class (```kubectl delete -f <path_to_secret_sync>```), and reapply the secret sync class (```kubectl apply -f <path_to_secret_sync>```). |
| `ControllerSpcError` | Secret update failed because the SSE failed to get the provider class or the provider class is misconfigured. | Review the provider class and correct any errors. Then, delete the `SecretSync` object (```kubectl delete secretsync <secret-name>```), delete the provider class (```kubectl delete -f <path_to_provider>```), and reapply the provider class (```kubectl apply -f <path_to_provider>```). |
| `ControllerInternalError` | Secret update failed due to an internal error in the SSE. | Check the SSE logs or the events for more information: <br>```kubectl get pods -n azure-secret-store``` <br>```kubectl logs <secret-sync-controller-pod-name> -n azure-secret-store --container='manager'``` |
| `SecretPatchFailedUnknownError` | Secret update failed during patching the Kubernetes secret value. This failure might occur if the secret was modified by someone other than the SSE or if there were issues during an update of the SSE. | Try deleting the secret and `SecretSync` object, then let the SSE recreate the secret by reapplying the secret sync CR: <br>```kubectl delete secret <secret-name>``` <br>```kubectl delete secretsync <secret-name>```  <br>```kubectl apply -f <path_to_secret_sync>``` |

## Remove the SSE

To remove the SSE and stop synchronizing secrets, uninstall it with the `az k8s-extension delete` command:

```console
az k8s-extension delete --name ssarcextension --cluster-name $CLUSTER_NAME  --resource-group $RESOURCE_GROUP  --cluster-type connectedClusters    
```

Uninstalling the extension doesn't remove secrets, `SecretSync` objects, or CRDs from the cluster. These objects must be removed directly with `kubectl`.

Deleting the SecretSync CRD removes all `SecretSync` objects, and by default removes all owned secrets, but secrets may persist if:

- You modified ownership of any of the secrets.
- You changed the [garbage collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/) settings in your cluster, including setting different [finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/).

In these cases, secrets must be deleted directly using `kubectl`.

## Next steps

- Learn more about [Azure Arc extensions](extensions.md).
- Learn more about [Azure Key Vault](/azure/key-vault/general/overview).
