# Azure Key Vault Provider for Secrets Store CSI Driver

[![Build Status](https://dev.azure.com/azure/secrets-store-csi-driver-provider-azure/_apis/build/status/secrets-store-csi-driver-provider-azure-ci?branchName=master)](https://dev.azure.com/azure/secrets-store-csi-driver-provider-azure/_build/latest?definitionId=67&branchName=master)

Azure Key Vault provider for [Secrets Store CSI driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver) allows you to get secret contents stored in an [Azure Key Vault](https://docs.microsoft.com/en-us/azure/key-vault/general/overview) instance and use the Secrets Store CSI driver interface to mount them into Kubernetes pods.

## Features

- Mounts secrets/keys/certs on pod start using a CSI volume
- Supports mounting multiple secrets store objects as a single volume
- Supports pod identity to restrict access with specific identities
- Supports pod portability with the SecretProviderClass CRD
- Supports windows containers (Kubernetes version v1.18+)
- Supports sync with Kubernetes Secrets (Secrets Store CSI Driver v0.0.10+)
- Supports multiple secrets stores providers in the same cluster.

#### Table of Contents

- [Demo](#demo)
- [Usage](#usage)
  - [Install with Helm](#install-the-secrets-store-csi-driver-and-the-azure-keyvault-provider)
  - [Using the Azure Key Vault Provider](#using-the-azure-key-vault-provider)
- [Azure Key Vault Provider Features](#azure-key-vault-provider-features)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [Testing](#testing)
- [Support](#support)

## Demo

_WIP_

## Usage

This guide will walk you through the steps to configure and run the Azure Key Vault provider for Secrets Store CSI driver on Kubernetes.

### Install the Secrets Store CSI Driver and the Azure Keyvault Provider
**Prerequisites**

Recommended Kubernetes version:
- For Linux - v1.16.0+
- For Windows - v1.18.0+

> For Kubernetes version 1.15 and below, please use [Azure Keyvault Flexvolume](https://github.com/Azure/kubernetes-keyvault-flexvol)

**Deployment using Helm**

Follow [this guide](charts/csi-secrets-store-provider-azure/README.md) to install the Secrets Store CSI driver and the Azure Key Vault provider using Helm.

Alternatively, follow [this guide](docs/install-yamls.md) to install using deployment yamls.

### Using the Azure Key Vault Provider

#### Create a new Azure Key Vault resource or use an existing one

In addition to a Kubernetes cluster, you will need an Azure Key Vault resource with secret content to access. Follow [this quickstart tutorial](https://docs.microsoft.com/azure/key-vault/secrets/quick-create-portal) to deploy an Azure Key Vault and add an example secret to it.

Review the settings you desire for your Key Vault, such as what resources (Azure VMs, Azure Resource Manager, etc.) and what kind of network endpoints can access secrets in it.

Take note of the following properties for use in the next section:

1. Name of secret object in Key Vault
1. Secret content type (secret, key, cert)
1. Name of Key Vault resource
1. Resource group the Key Vault resides within
1. Azure Subscription ID the Key Vault was provisioned with
1. Azure Tenant ID the Subscription ID belongs to

#### Create your own SecretProviderClass Object

Create a `SecretProviderClass` custom resource to provide provider-specific parameters for the Secrets Store CSI driver. In this example, use an existing Azure Key Vault or the Azure Key Vault resource created previously.

Update [this sample deployment](examples/v1alpha1_secretproviderclass.yaml) to create a `SecretProviderClass` resource to provide Azure-specific parameters for the Secrets Store CSI driver.

To provide identity to access key vault, refer to the following [section](#provide-identity-to-access-key-vault).

  ```yaml
  apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
  kind: SecretProviderClass
  metadata:
    name: azure-kvname
  spec:
    provider: azure                   
    parameters:
      usePodIdentity: "false"         # [OPTIONAL for Azure] if not provided, will default to "false"
      useVMManagedIdentity: "false"   # [OPTIONAL available for version > 0.0.4] if not provided, will default to "false"
      userAssignedIdentityID: "client_id"  # [OPTIONAL available for version > 0.0.4] use the client id to specify which user assigned managed identity to use. If using a user assigned identity as the VM's managed identity, then specify the identity's client id. If empty, then defaults to use the system assigned identity on the VM
      keyvaultName: "kvname"          # the name of the KeyVault
      cloudName: ""          # [OPTIONAL available for version > 0.0.4] if not provided, azure environment will default to AzurePublicCloud
      objects:  |
        array:
          - |
            objectName: secret1
            objectAlias: SECRET_1     # [OPTIONAL available for version > 0.0.4] object alias
            objectType: secret        # object types: secret, key or cert
            objectVersion: ""         # [OPTIONAL] object versions, default to latest if empty
          - |
            objectName: key1
            objectAlias: ""
            objectType: key
            objectVersion: ""
      resourceGroup: "rg1"            # [REQUIRED for version < 0.0.4] the resource group of the KeyVault
      subscriptionId: "subid"         # [REQUIRED for version < 0.0.4] the subscription ID of the KeyVault
      tenantId: "tid"                 # the tenant ID of the KeyVault

  ```

  | Name                   | Required | Description                                                     | Default Value |
  | -----------------------| -------- | --------------------------------------------------------------- | ------------- |
  | provider               | yes      | specify name of the provider                                    | ""            |
  | usePodIdentity         | no       | specify access mode: service principal or pod identity          | "false"       |
  | useVMManagedIdentity   | no       | [__*available for version > 0.0.4*__] specify access mode to enable use of VM's managed identity    |  "false"|
  | userAssignedIdentityID | no       | [__*available for version > 0.0.4*__] the user assigned identity ID is required for VMSS User Assigned Managed Identity mode  | ""       |
  | keyvaultName           | yes      | name of a Key Vault instance                                    | ""            |
  | cloudName              | no       | [__*available for version > 0.0.4*__] name of the azure cloud based on azure go sdk (AzurePublicCloud,AzureUSGovernmentCloud, AzureChinaCloud, AzureGermanCloud)| "" |
  | objects                | yes      | a string of arrays of strings                                   | ""            |
  | objectName             | yes      | name of a Key Vault object                                      | ""            |
  | objectAlias            | no       | [__*available for version > 0.0.4*__] specify the filename of the object when written to disk - defaults to objectName if not provided | "" |
  | objectType             | yes      | type of a Key Vault object: secret, key or cert                 | ""            |
  | objectVersion          | no       | version of a Key Vault object, if not provided, will use latest | ""            |
  | resourceGroup          | no      | [__*required for version < 0.0.4*__] name of resource group containing key vault instance            | ""            |
  | subscriptionId         | no      | [__*required for version < 0.0.4*__] subscription ID containing key vault instance                   | ""            |
  | tenantId               | yes      | tenant ID containing key vault instance                         | ""            |

#### Provide Identity to Access Key Vault

The Azure Key Vault Provider offers four modes for accessing a Key Vault instance:

1. [Service Principal](docs/service-principal-mode.md)
1. [Pod Identity](docs/pod-identity-mode.md)
1. [VMSS User Assigned Managed Identity](docs/user-assigned-msi-mode.md)
1. [VMSS System Assigned Managed Identity](docs/system-assigned-msi-mode.md)

#### Update your Deployment Yaml

To ensure your application is using the Secrets Store CSI driver, update your deployment yaml to use the `secrets-store.csi.k8s.io` driver and reference the `SecretProviderClass` resource created in the previous step.

Update your [linux deployment yaml](examples/nginx-pod-secrets-store-inline-volume-secretproviderclass.yaml) or [windows deployment yaml](examples/windows-pod-secrets-store-inline-volume-secret-providerclass.yaml) to use the Secrets Store CSI driver and reference the `SecretProviderClass` resource created in the previous step. 
    
  ```yaml
    volumes:
      - name: secrets-store-inline
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "azure-kvname"
  ```

#### Deploy your Kubernetes Resources

  1. Deploy the SecretProviderClass yaml created previously. For example:

     `kubectl apply -f ./examples/v1alpha1_secretproviderclass.yaml`

  1. Deploy the application yaml created previously. For example:

     `kubectl apply -f ./examples/nginx-pod-secrets-store-inline-volume-secretproviderclass.yaml`

#### Validate the secret

To validate, once the pod is started, you should see the new mounted content at the volume path specified in your deployment yaml.

  ```bash
  ## show secrets held in secrets-store
  kubectl exec -it nginx-secrets-store-inline ls /mnt/secrets-store/

  ## print a test secret held in secrets-store
  kubectl exec -it nginx-secrets-store-inline cat /mnt/secrets-store/secret1

  ```

## Azure Key Vault Provider Features

### Secret Content is Mounted on Pod Start
On pod start and restart, the driver will call the Azure provider binary to retrieve the secret content from the Azure Key Vault instance you have specified in the `SecretProviderClass` custom resource. Then the content will be mounted to the container's file system. 

To validate, once the pod is started, you should see the new mounted content at the volume path specified in your deployment yaml.

```bash
kubectl exec -it nginx-secrets-store-inline ls /mnt/secrets-store/
foo
```

### [OPTIONAL] Sync with Kubernetes Secrets

In some cases, you may want to create a Kubernetes Secret to mirror the mounted content. Use the optional `secretObjects` field to define the desired state of the synced Kubernetes secret objects.

> NOTE: Make sure the `objectName` in `secretObjects` matches the name of the mounted content. This could be the object name or the object alias.

A `SecretProviderClass` custom resource should have the following components:
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: my-provider
spec:
  provider: azure                             
  secretObjects:                              # [OPTIONAL] SecretObject defines the desired state of synced K8s secret objects
  - data:
    - key: username                           # data field to populate
      objectName: foo1                        # name of the mounted content to sync. this could be the object name or the object alias
    secretName: foosecret                     # name of the Kubernetes Secret object
    type: Opaque                              # type of the Kubernetes Secret object e.g. Opaque, kubernetes.io/tls
```
> NOTE: Here is the list of supported Kubernetes Secret types: `Opaque`, `kubernetes.io/basic-auth`, `bootstrap.kubernetes.io/token`, `kubernetes.io/dockerconfigjson`, `kubernetes.io/dockercfg`, `kubernetes.io/ssh-auth`, `kubernetes.io/service-account-token`, `kubernetes.io/tls`.  

Here is a sample [`SecretProviderClass` custom resource](https://github.com/kubernetes-sigs/secrets-store-csi-driver/blob/master/test/bats/tests/azure_synck8s_v1alpha1_secretproviderclass.yaml) that syncs a secret from Azure Key Vault to a Kubernetes secret.

### [OPTIONAL] Set ENV VAR

Once the secret is created, you may wish to set an ENV VAR in your deployment to reference the new Kubernetes secret.

```yaml
spec:
  containers:
  - image: nginx
    name: nginx
    env:
    - name: SECRET_USERNAME
      valueFrom:
        secretKeyRef:
          name: foosecret
          key: username
```
Here is a sample [deployment yaml](https://github.com/kubernetes-sigs/secrets-store-csi-driver/blob/master/test/bats/tests/nginx-deployment-synck8s-azure.yaml) that creates an ENV VAR from the synced Kubernetes secret.


## Troubleshooting

To troubleshoot issues with the csi driver and the provider, you can look at logs from the `secrets-store` container of the csi driver pod running on the same node as your application pod:

  ```bash
  kubectl get pod -o wide
  # find the secrets store csi driver pod running on the same node as your application pod

  kubectl logs csi-secrets-store-secrets-store-csi-driver-7x44t secrets-store
  ```

## Contributing

Please refer to [CONTRIBUTING.md](./CONTRIBUTING.md) for more information.

## Testing

For documentation on how to locally test the Secrets Store CSI Driver Provider for Azure, please refer to [this guide](docs/testing.md)

## Support

Azure Key Vault Provider for Secrets Store CSI Driver is an open source project that is [**not** covered by the Microsoft Azure support policy](https://support.microsoft.com/en-us/help/2941892/support-for-linux-and-open-source-technology-in-azure). [Please search open issues here](https://github.com/Azure/secrets-store-csi-driver-provider-azure/issues), and if your issue isn't already represented please [open a new one](https://github.com/Azure/secrets-store-csi-driver-provider-azure/issues/new/choose). The project maintainers will respond to the best of their abilities.
