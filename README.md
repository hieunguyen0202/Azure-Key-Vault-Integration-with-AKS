## Azure-Key-Vault-Integration-with-AKS

## Architecture

![Azure-Key-Vault-Integration-with-AKS drawio](https://github.com/hieunguyen0202/Azure-Key-Vault-Integration-with-AKS/assets/98166568/c7692ea0-c9c8-4e63-80b7-3a7b8578e88b)

## Implementation
### AKS setup using CLI
- List Resource Group
  
```
az group list --output table
```
- Create Azure Resource Group

```
az group create --name myResourceGroup --location eastus2
```

### AKS Creation and Configuration

- Create an AKS cluster with Azure Key Vault provider for Secrets Store CSI Driver support

```
az aks create --name myAKSCluster --resource-group myResourceGroup --node-count 1 --enable-addons azure-keyvault-secrets-provider --enable-oidc-issuer --enable-workload-identity --generate-ssh-keys
```

- Get the Kubernetes cluster credentials (Update kubeconfig)

```
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

- Verify that each node in your cluster's node pool has a Secrets Store CSI Driver pod and a Secrets Store Provider Azure pod running

```
kubectl get pods -n kube-system -l 'app in (secrets-store-csi-driver,secrets-store-provider-azure)' -o wide
```

### Keyvault creation and configuration

- Create a key vault with Azure role-based access control (Azure RBAC).

```
az keyvault create -n aks-demo-keyvault -g myResourceGroup -l eastus2 --enable-rbac-authorization
```
```
## Create a new Azure key vault
az keyvault create --name <keyvault-name> --resource-group myResourceGroup --location eastus2 --enable-rbac-authorization

## Update an existing Azure key vault
az keyvault update --name <keyvault-name> --resource-group myResourceGroup --location eastus2 --enable-rbac-authorization
```
- To view access. go to `Access control (IAM)` -> Click on `Add role assignment` -> Choose `Key Vault Administrator`
- And assign access to `User, group, or service principal` -> Select a approriate member
- Create a new `key` and `secret`

### Connect your Azure ID to the Azure Key Vault Secrets Store CSI Driver 

- Configure workload identity

```
export SUBSCRIPTION_ID=4078f7a0-8adc-47c8-9691-309ad92b124a
export RESOURCE_GROUP=myResourceGroup
export UAMI=azurekeyvaultsecretsprovider-keyvault-demo-cluster
export KEYVAULT_NAME=aks-demo-keyvault
export CLUSTER_NAME=myAKSCluster

az account set --subscription $SUBSCRIPTION_ID
```
- Create a managed identity

```
az identity create --name $UAMI --resource-group $RESOURCE_GROUP

export USER_ASSIGNED_CLIENT_ID="$(az identity show -g $RESOURCE_GROUP --name $UAMI --query 'clientId' -o tsv)"
export IDENTITY_TENANT=$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query identity.tenantId -o tsv)
```

- Create a role assignment that grants the workload ID access the key vault (Assign role for managed identity)

```
export KEYVAULT_SCOPE=$(az keyvault show --name $KEYVAULT_NAME --query id -o tsv)

az role assignment create --role "Key Vault Administrator" --assignee $USER_ASSIGNED_CLIENT_ID --scope $KEYVAULT_SCOPE
```

- Get the AKS cluster OIDC Issuer URL 

```
export AKS_OIDC_ISSUER="$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --query "oidcIssuerProfile.issuerUrl" -o tsv)"
echo $AKS_OIDC_ISSUER
```

- Create the service account for the pod

```
export SERVICE_ACCOUNT_NAME="workload-identity-sa"
export SERVICE_ACCOUNT_NAMESPACE="default" 
```

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  name: ${SERVICE_ACCOUNT_NAME}
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
EOF
```
- Check status

```
kubectl get sa
```
- Setup Federation

```
export FEDERATED_IDENTITY_NAME="aksfederatedidentity" 

az identity federated-credential create --name $FEDERATED_IDENTITY_NAME --identity-name $UAMI --resource-group $RESOURCE_GROUP --issuer ${AKS_OIDC_ISSUER} --subject system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}
```

- Create the Secret Provider Class

```
cat <<EOF | kubectl apply -f -
# This is a SecretProviderClass example using workload identity to access your key vault
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname-wi # needs to be unique per namespace
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "${USER_ASSIGNED_CLIENT_ID}" # Setting this to use workload identity
    keyvaultName: ${KEYVAULT_NAME}       # Set to the name of your key vault
    cloudName: ""                         # [OPTIONAL for Azure] if not provided, the Azure environment defaults to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: secret1             # Set to the name of your secret
          objectType: secret              # object types: secret, key, or cert
          objectVersion: ""               # [OPTIONAL] object versions, default to latest if empty
        - |
          objectName: key1                # Set to the name of your key
          objectType: key
          objectVersion: ""
    tenantId: "${IDENTITY_TENANT}"        # The tenant ID of the key vault
EOF
```
### Verify Keyvault AKS Integration

- Create a sample pod to mount the secrets

```
cat <<EOF | kubectl apply -f -
# This is a sample pod definition for using SecretProviderClass and workload identity to access your key vault
kind: Pod
apiVersion: v1
metadata:
  name: busybox-secrets-store-inline-wi
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: "workload-identity-sa"
  containers:
    - name: busybox
      image: registry.k8s.io/e2e-test-images/busybox:1.29-4
      command:
        - "/bin/sleep"
        - "10000"
      volumeMounts:
      - name: secrets-store01-inline
        mountPath: "/mnt/secrets-store"
        readOnly: true
  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-kvname-wi"
EOF
```

- List the contents of the volume

```
kubectl exec busybox-secrets-store-inline-wi -- ls /mnt/secrets-store/
```

- Verify the contents in the file

```
kubectl exec busybox-secrets-store-inline-wi -- cat /mnt/secrets-store/secret1
```
### Delete Everything

```
az group delete --name myResourceGroup
```


                             +------------------------------------------------------+
                             |                    Azure Environment                  |
                             +------------------------------------------------------+
                                                |                         |
                                                |                         |
                            +-------------------+                         +-------------------+
                            |                                        |                      |
                            v                                        v                      v
            +------------------------------+       +------------------------------+       +------------------------------+
            |          Office 1             |       |          VPN Gateway         |       |          Office 2             |
            |       (Location 1)            |       +------------------------------+       |       (Location 2)            |
            |  VNet: Office1VNet (10.1.0.0/16)|<---->|     VPN Gateway Connection   |<----->|  VNet: Office2VNet (10.2.0.0/16)|
            +------------------------------+       | (VPN: Site-to-Site between    |       +------------------------------+
                |          |          |            |    Office1VNet & Office2VNet) |            |          |          |
                v          v          v            +------------------------------+            v          v          v
    +---------------+ +---------------+ +---------------+                                      +---------------+ +---------------+ +---------------+
    | DevSubnet     | | UATSubnet1    | | UATSubnet2    |                                      | DevSubnet     | | UATSubnet1    | | UATSubnet2    |
    | (10.1.1.0/24) | | (10.1.2.0/24) | | (10.1.3.0/24) |                                      | (10.2.1.0/24) | | (10.2.2.0/24) | | (10.2.3.0/24) |
    +---------------+ +---------------+ +---------------+                                      +---------------+ +---------------+ +---------------+
          |               |               |                                                       |               |               |
          v               v               v                                                       v               v               v
    +------------+   +------------+   +------------+                                         +------------+   +------------+   +------------+
    | Dev-VM-1   |   | UAT1-VM-1  |   | UAT2-VM-1  |                                         | Dev-VM-2   |   | UAT1-VM-2  |   | UAT2-VM-2  |
    +------------+   +------------+   +------------+                                         +------------+   +------------+   +------------+
      (10.1.1.4)       (10.1.2.4)       (10.1.3.4)                                             (10.2.1.4)       (10.2.2.4)       (10.2.3.4)

                +-------------------------------+                                    +-------------------------------+
                | FileShareSubnet                |                                    | FileShareSubnet                |
                | (10.1.5.0/24)                  |                                    | (10.2.5.0/24)                  |
                +-------------------------------+                                    +-------------------------------+
                                |                                                                  |
                                v                                                                  v
                    +----------------+                                                    +----------------+
                    | FileShare-VM-1 |                                                    | FileShare-VM-2 |
                    +----------------+                                                    +----------------+
                      (10.1.5.4)                                                            (10.2.5.4)


                             +-------------------------+             +-------------------------+
                             |    Network Security     |             |    Network Security     |
                             |    NSG DevSubnet        |             |    NSG UATSubnet1       |
                             +-------------------------+             +-------------------------+
                                 |            |                         |            |
                                 v            v                         v            v
                               Rule 1       Rule 2                    Rule 1       Rule 2
                               Allow        Deny                      Allow        Deny

                             +-------------------------+             +-------------------------+
                             |    Network Security     |             |    Network Security     |
                             |    NSG UATSubnet2       |             |    NSG FileShareSubnet  |
                             +-------------------------+             +-------------------------+
                                 |            |                         |            |
                                 v            v                         v            v
                               Rule 1       Rule 2                    Rule 1       Rule 2
                               Allow        Deny                      Allow        Deny

                           +-----------------------------------------------------+
                           |        Role-Based Access Control (RBAC)            |
                           +-----------------------------------------------------+
                               |                |                |
                               v                v                v
                           System Dept      Dev Dept         HR Dept
                            (Full Access)   (VM Access)      (FileShare Access)
