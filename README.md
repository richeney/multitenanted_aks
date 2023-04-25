# Multi-tenanted AKS POC using workload identity

**Work in progress!!!**

AKS cluster with

- OIDC
- workload identity support
- Azure CNI Overlay
- Calico network policy

Per customer resources

- Kubernetes namespace
- workload identity
- user assigned managed identity
- Azure Key Vault

## Prereqs

Need minimum of Azure CLI 2.47.0 with aks-preview extension. Bash assumed - tested on Ubuntu in WSL2.

```bash
az feature register --namespace "Microsoft.ContainerService" --name "AzureOverlayPreview"
```

Check registered:

```bash
az feature show --namespace "Microsoft.ContainerService" --name "AzureOverlayPreview"
```

Then refresh the provider:

```bash
az provider register --namespace Microsoft.ContainerService
```

## Variables

Use "akswi" as a prefix throughout. Use first 8 characters of the subscription in FQDNs.

```bash
export PREFIX="richeney"
export AKS_CLUSTER="${PREFIX}aks"
export AZURE_DEFAULTS_GROUP=${AKS_CLUSTER}
export AZURE_DEFAULTS_LOCATION="uksouth"
export SUBSCRIPTION="$(az account show --query id --output tsv)"
export SUBSTR=$(cut -f1 -d\- <<<${SUBSCRIPTION})
```

## Create AKS cluster

```bash
az group create --name "${AZURE_DEFAULTS_GROUP}"

az network vnet create  --name "${AKS_CLUSTER}-vnet" --address-prefixes "172.17.1.0/24" \
  --subnet-name "${AKS_CLUSTER}-node-subnet" --subnet-prefixes 172.17.1.0/27

SUBNET_ID=$(az network vnet subnet show --name "${AKS_CLUSTER}-node-subnet" --vnet-name "${AKS_CLUSTER}-vnet" --query id -otsv)

az aks create --name "${AKS_CLUSTER}" \
  --node-count 3 --zones 1 2 3 \
  --enable-oidc-issuer --enable-workload-identity \
  --network-plugin azure --network-policy calico \
  --network-plugin-mode overlay --pod-cidr "10.244.0.0/16" \
  --vnet-subnet-id "${SUBNET_ID}"

  ## --enable-azure-rbac --aad-admin-group-object-ids --attach-acr


export AKS_OIDC_ISSUER="$(az aks show --name ${AKS_CLUSTER} --query "oidcIssuerProfile.issuerUrl" -otsv)"

az aks get-credentials --name ${AKS_CLUSTER}
```

## Create resources for customers

## **YOU ARE HERE - enable RBAC on the KV**

```bash
for customer in alpha beta
do
  CUSTOMER="${AZURE_DEFAULTS_GROUP}-$customer"
  KEY_VAULT_NAME="${AZURE_DEFAULTS_GROUP}-${SUBSTR}-${CUSTOMER}
  az group create --name "${CUSTOMER}"
  az identity create --name "${CUSTOMER}" --resource-group "${CUSTOMER}"
  az keyvault create --name "${KEY_VAULT_NAME}" --resource-group "${CUSTOMER}"
  az keyvault secret set --vault-name "${KEY_VAULT_NAME}" --name "customer-name" --value "${customer}"
  export USER_ASSIGNED_CLIENT_ID="$(az identity show --name "${CUSTOMER}" --resource-group "${CUSTOMER}" --query 'clientId' -otsv)"
done