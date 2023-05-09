# Multi-tenanted AKS POC using workload identity

**Work in progress!!!**

AKS cluster with

- OIDC
- workload identity support (preview)
- Web app routing (preview)
  - Azure Key Vault Provider for Secrets Store CSI Driver support
  - Azure Key Vault with self signed cert
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

    Needed with web application routing?

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

Set PREFIX to ensure unique Azure resource FQDNs, e.g. key vaults. DNS_HOSTNAME will be used for Web Application Routing, DNS Zones and the self signed SSL certificate.

```bash
export PREFIX="richeney"
export DNS_ZONE="azurecitadel.com"
export AZURE_DEFAULTS_LOCATION="uksouth"
export CUSTOMERS="alpha beta"
```

Create the SSL files.

```bash
openssl req -new -x509 -nodes -out aks-ingress-tls.crt -keyout aks-ingress-tls.key \
  -subj "/CN=${DNS_HOSTNAME}" -addext "subjectAltName=DNS:${DNS_HOSTNAME}"

openssl pkcs12 -export -in aks-ingress-tls.crt -inkey aks-ingress-tls.key \
  -out aks-ingress-tls.pfx
```

> ⚠️ Only hit enter when prompted for a password if you are running a test cluster.

Derived additional variables.

```bash
export AKS_CLUSTER="${PREFIX}aks"
export AZURE_DEFAULTS_GROUP=${AKS_CLUSTER}
export SUBSCRIPTION="$(az account show --query id --output tsv)"
export KEY_VAULT_NAME="${AKS_CLUSTER}-kv"
```

## Create AKS cluster

### Resource group and resources

```bash
az group create --name "${AZURE_DEFAULTS_GROUP}"

az network nsg create --name "${AKS_CLUSTER}-node-subnet-nsg"

az network nsg rule create --name "AllowHTTPandHTTPS" \
  --nsg-name "${AKS_CLUSTER}-node-subnet-nsg" --priority 101 \
  --destination-port-ranges 80 8080 443 8443 --protocol Tcp

az network vnet create --name "${AKS_CLUSTER}-vnet" --address-prefixes "172.17.1.0/24"

az network vnet subnet create --name "${AKS_CLUSTER}-node-subnet" \
  --vnet-name "${AKS_CLUSTER}-vnet" --address-prefixes 172.17.1.0/27 \
  --network-security-group "${AKS_CLUSTER}-node-subnet-nsg"

SUBNET_ID=$(az network vnet subnet show --name "${AKS_CLUSTER}-node-subnet" --vnet-name "${AKS_CLUSTER}-vnet" --query id -otsv)

az keyvault create --name ${KEY_VAULT_NAME} --retention-days 7
az keyvault certificate import --vault-name ${KEY_VAULT_NAME} -n "${AKS_CLUSTER}-tls" -f aks-ingress-tls.pfx
```

## DNS Zone

```bash
DNS_ZONE_RG="azurecitadel"
```

Reuse your existing one if it exists. If not, create.

```bash
az network dns zone create --name ${DNS_ZONE} --resource-group ${DNS_ZONE_RG}
```

Grab the DNS Zone's resource ID.

```bash
DNS_ZONE_ID=$(az network dns zone show --name ${DNS_ZONE} --resource-group ${DNS_ZONE_RG} --query id -otsv)
```

If you need to update
az aks addon update --name ${AKS_CLUSTER} --addon web_application_routing --dns-zone-resource-id=${DNS_ZONE_ID}
```

### Create the AKS cluster itself

```bash
az aks create --name "${AKS_CLUSTER}" \
  --node-count 1 --zones 1 2 3 \
  --enable-oidc-issuer --enable-workload-identity \
  --enable-addons azure-keyvault-secrets-provider,web_application_routing \
  --enable-secret-rotation --dns-zone-resource-id=${DNS_ZONE_ID} \
  --network-plugin azure --network-policy calico \
  --network-plugin-mode overlay --pod-cidr "10.244.0.0/16" \
  --vnet-subnet-id "${SUBNET_ID}" \
  --generate-ssh-keys
```

Assign the DNS ZONE Contributor role and a Key Vault access policy to read certs and secrets.

```bash
WEBAPPROUTING_OBJECT_ID=$(az aks show --name ${AKS_CLUSTER} --query ingressProfile.webAppRouting.identity.objectId -o tsv)
az role assignment create --role "DNS Zone Contributor" --assignee ${WEBAPPROUTING_OBJECT_ID} --scope ${DNS_ZONE_ID}
az keyvault set-policy --name ${KEY_VAULT_NAME} --object-id ${WEBAPPROUTING_OBJECT_ID} --secret-permissions get --certificate-permissions get
```


Check that the cluster is in Azure CNI Overlay mode and using the manually created virtual network:

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

kubectl exec --stdin --tty <containername> -- /bin/bash
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


## Web Application Routing add-on

Preview, but recommended. Switch.

<https://learn.microsoft.com/azure/aks/web-app-routing?tabs=without-osm>

The Web Application Routing add-on deploys the following components:

* **nginx ingress controller**: This ingress controller is exposed to the internet.
* **external-dns controller**: This controller watches for Kubernetes ingress resources and creates DNS A records in the cluster-specific DNS zone. This is only deployed when you pass in the --dns-zone-resource-id argument.
