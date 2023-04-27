# Multi-tenanted AKS POC using workload identity

**Work in progress!!!**

AKS cluster with

- OIDC
- workload identity support
- Azure Key Vault Provider for Secrets Store CSI Driver support
- Azure CNI Overlay
- Calico network policy

Per customer resources

- Kubernetes namespace
- workload identity
- user assigned managed identity
- Azure Key Vault

## Prereqs

### Bash

Bash environment is assumed. Tested on Ubuntu in WSL2.

- [Azure CLI](https://aka.ms/installtheazurecli)
  - Minimum version is 2.47.0
  - aks-preview extension

    ```bash
    az extension add --name aks-preview
    ```

- kubectl

    ```bash
    sudo az aks install-cli
    ```

- [kubectx](https://github.com/ahmetb/kubectx) (optional)

    Enables fuzzy `kubectl ctx` and `kubectl ns` to switch context and namespace.

    Install [krew](https://krew.sigs.k8s.io/docs/user-guide/setup/install) first.

    Use krew to install the two extensions, and add [fzf](https://github.com/junegunn/fzf#installation).

    ```bash
    kubectl krew install ctx
    kubectl krew install ns
    sudo apt install fzf
    ```

- [Helm](https://helm.sh/docs/intro/install)

    ```bash
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh
    ```

### Register AzureOverlayPreview feature

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
export CUSTOMERS="alpha beta"
```

## Create AKS cluster

```bash
az group create --name "${AZURE_DEFAULTS_GROUP}"

az network vnet create  --name "${AKS_CLUSTER}-vnet" --address-prefixes "172.17.1.0/24" \
  --subnet-name "${AKS_CLUSTER}-node-subnet" --subnet-prefixes 172.17.1.0/27

SUBNET_ID=$(az network vnet subnet show --name "${AKS_CLUSTER}-node-subnet" --vnet-name "${AKS_CLUSTER}-vnet" --query id -otsv)
```

Create the cluster

```bash
az aks create --name "${AKS_CLUSTER}" \
  --node-count 3 --zones 1 2 3 \
  --enable-oidc-issuer --enable-workload-identity \
  --enable-addons azure-keyvault-secrets-provider \
  --network-plugin azure --network-policy calico \
  --network-plugin-mode overlay --pod-cidr "10.244.0.0/16" \
  --vnet-subnet-id "${SUBNET_ID}"
```

Check that the cluster is un Azure CNI Overlay mode and using the manually created virtual network:

```bash
az aks show --name "${AKS_CLUSTER}" --query networkProfile --output yamlc
```

Merge credentials into ~/.kube/config.

```bash
az aks get-credentials --name "${AKS_CLUSTER}"
```

Check cluster access.

```bash
kubectl get nodes
```

Example output.

```text
NAME                                STATUS     ROLES   AGE     VERSION
aks-nodepool1-24813231-vmss000000   NotReady   agent   2d20h   v1.25.6
aks-nodepool1-24813231-vmss000001   NotReady   agent   2d20h   v1.25.6
aks-nodepool1-24813231-vmss000002   NotReady   agent   2d20h   v1.25.6
```

## Create customer resources

Grab the OpenID Connect issuer URL.

```bash
export AKS_OIDC_ISSUER="$(az aks show --name "${AKS_CLUSTER}" --query "oidcIssuerProfile.issuerUrl" -otsv)"
```

Create the Azure resources

```bash
for customer in ${CUSTOMERS:-alpha beta}
do
  (
    az group create --name "${AZURE_DEFAULTS_GROUP}-${customer}"
    identity="${AZURE_DEFAULTS_GROUP}-${customer}"

    export AZURE_DEFAULTS_GROUP="${AZURE_DEFAULTS_GROUP}-${customer}"

    az identity create --name "${identity}"

    az keyvault create --name "${identity}-kv" --retention-days 7
    vault_uri=$(az keyvault show --name "${identity}-kv" --query properties.vaultUri -otsv)

    client_id=$(az identity show --name "${identity}" --query clientId -otsv)
    az keyvault set-policy --name "${identity}-kv" --secret-permissions get --spn ${client_id}

    az keyvault secret set --vault-name "${identity}-kv" --name "my-secret" --value "Hello ${customer}!"

    az identity federated-credential create --name "${identity}-aad" \
      --identity-name "${identity}" \
      --issuer "${AKS_OIDC_ISSUER}" \
      --subject system:serviceaccount:"${customer}":"${identity}" \
      --audience api://AzureADTokenExchange
  )
done
```

> The parentheses means each loop iteration runs in a subshell. Any variables (or env var changes) will be discarded at the end of the loop. Changing AZURE_DEFAULTS_GROUP only defaults --resource-group to the customer one for the duration of the loop iteration.

## Generate kubernetes yaml files

This section creates a directory for each customer, plus yaml files for the service account and a simple pod that uses the MSAL library in Go. ([Source code](https://github.com/Azure/azure-workload-identity/blob/main/examples/msal-go/main.go).)

```bash
for customer in ${CUSTOMERS:-alpha beta}
do
  (
    kubectl create namespace $customer
    identity="$AZURE_DEFAULTS_GROUP-$customer"
    export AZURE_DEFAULTS_GROUP=$identity
    client_id=$(az identity show --name $identity --query clientId -otsv)
    vault_uri=$(az keyvault show --name $identity-kv --query properties.vaultUri -otsv)

    [[ -d $customer ]] || mkdir $customer
    cd $customer

    cat > service_account.yml <<EOF1
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: "${client_id}"
  name: "${identity}"
  namespace: "${customer}"
EOF1

    cat > pod.yml" <<EOF2
apiVersion: v1
kind: Pod
metadata:
  name: msal-go
  namespace: ${customer}
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: ${identity}
  containers:
    - image: ghcr.io/azure/azure-workload-identity/msal-go
      name: oidc
      env:
      - name: KEYVAULT_URL
        value: ${vault_uri}
      - name: SECRET_NAME
        value: my-secret
  nodeSelector:
    kubernetes.io/os: linux
EOF2


  kubectl apply -f service_account.yml"
  kubectl apply -f pod.yml"
  )
done
```

## Check the logs for a pod

kubectl config set-context --current --namespace=alpha
kubectl get pods
kubectl describe pod msal-go

```text
Environment:
  KEYVAULT_URL:                https://richeneyaks-beta-kv.vault.azure.net/
  SECRET_NAME:                 my-secret
  AZURE_CLIENT_ID:             cf37a6f2-8e00-494e-831f-59b1823f82da
  AZURE_TENANT_ID:             3c584bbd-915f-4c70-9f2e-7217983f22f6
  AZURE_FEDERATED_TOKEN_FILE:  /var/run/secrets/azure/tokens/azure-identity-token
  AZURE_AUTHORITY_HOST:        https://login.microsoftonline.com/
```

Note that Azure workload identity has pulled through additional env vars via the ServiceAccount.

```bash
kubectl logs msal-go
```

The pod logs the secret value from the Key Vault once per minute. Below is the live log as seen from the Azure Portal

## Secrets Store CSI Driver

Install the [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/getting-started/installation.html)

```bash
helm repo add csi-secrets-store-provider-azure https://azure.github.io/secrets-store-csi-driver-provider-azure/charts
helm install csi csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --namespace kube-system
```

kubectl get pods -l app=csi-secrets-store-provider-azure -A

NAMESPACE     NAME                                         READY   STATUS    RESTARTS   AGE
kube-system   csi-csi-secrets-store-provider-azure-cfvlk   1/1     Running   0          3m18s
kube-system   csi-csi-secrets-store-provider-azure-qf4gx   1/1     Running   0          3m18s
kube-system   csi-csi-secrets-store-provider-azure-zkswx   1/1     Running   0          3m18s

Add secrets

identity="richeneyaks-alpha"
az keyvault secret set --vault-name "${identity}-kv" --name "food" --value "aubergine"
az keyvault secret set --vault-name "${identity}-kv" --name "car" --value "Abarth"
az keyvault secret set --vault-name "${identity}-kv" --name "city" --value "Aberdeen"


kubectl get secretproviderclasses
kubectl describe secretproviderclasses richeneyaks-alpha-secrets

      envFrom:
        - secretRef:
            name: secret-info

kubectl exec --stdin --tty secret -- /bin/bash
grep ^ /mnt/secrets-store/*
env | grep APP
exit

kubectl exec -i -t secret -- /bin/sh -c 'grep ^ /mnt/secrets-store/*; env | grep APP'

## YADA

```bash
for customer in beta
do
  (
    export AZURE_DEFAULTS_GROUP=$AZURE_DEFAULTS_GROUP-$customer
    sql_server_name=richeneyaks-$customer-sql
    sql_db_name=mydb
    sql_username=azure
    sql_password=$(openssl rand -base64 10)  # 10-character random password
    echo $sql_password

    az sql server create --name $sql_server_name --admin-user "$sql_username" --admin-password "$sql_password"
    # az sql db create --name $sql_db_name --server $sql_server_name -g $rg -e Basic -c 5 --no-wait

    az sql db create --server $sql_server_name --name $sql_db_name \
      --edition GeneralPurpose --compute-model Serverless --family Gen5 \
      --min-capacity 0.5 --capacity 2 --auto-pause-delay 720

    sql_server_fqdn=$(az sql server show --name $sql_server_name --query fullyQualifiedDomainName -otsv)

    echo $sql_server_fqdn
  )
done
```

EjHl/fLxkZdASQ==
richeneyaks-beta-sql.database.windows.net

Example api.yml and web.yml - construct
Move secrets into keyvault

kubectl get pods
kubectl exec --stdin --tty api-57d84f95fd-gqzlk -- /bin/bash
curl http://api:8080/api/healthcheck
curl http://web

## Ingress

```bash
kubectl create namespace ingress-basic

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --create-namespace \
  --namespace ingress-basic \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

/git/multitenanted_aks/beta (main) $ kubectl get svc -ningress-basic
NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.0.14.42     51.142.209.17   80:30659/TCP,443:30122/TCP   36s
ingress-nginx-controller-admission   ClusterIP      10.0.112.111   <none>          443/TCP                      36s

```bash
kubectl create namespace ingress


```