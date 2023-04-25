# AKS Multi-tenanted - continue

Later - use YADA with private endpointed SQL

## Links

<https://learn.microsoft.com/en-us/azure/aks/azure-cni-overlay>
<https://learn.microsoft.com/en-us/azure/aks/hybrid/calico-networking-policy>
<https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview>
<https://learn.microsoft.com/en-us/azure/aks/learn/tutorial-kubernetes-workload-identity>
<https://learn.microsoft.com/en-us/azure/key-vault/general/quick-create-cli>
<https://github.com/microsoft/YADA>
<https://www.linkedin.com/pulse/use-workload-identity-preview-secrets-store-csi-arana-escobedo/>
<https://azure.github.io/secrets-store-csi-driver-provider-azure/docs/>
<https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-identity-access>

## Prereqs

Add kubectl to prereqs - `sudo az aks install-cli`

Recommend <https://github.com/ahmetb/kubectx>

* Install [krew](https://krew.sigs.k8s.io/docs/user-guide/setup/install/
* `kubectl krew install ctx`
* `kubectl krew install ns`
* [fzf](https://github.com/junegunn/fzf#installation) - `sudo apt install fzf`

Can then use `kubectl ctx` and `kubectl ns` to switch context and namespace.

## Variables

```bash
export AZURE_DEFAULTS_GROUP="richeneyaks"
export AZURE_DEFAULTS_LOCATION="uksouth"
export AKS_CLUSTER="richeneyaks"
```

## Continue

Check that the cluster is un Azure CNI Overlay mode and using the manually created virtual network:

```bash
az aks show --name $AKS_CLUSTER --query networkProfile --output yamlc
```

Example output:

```yaml
dnsServiceIp: 10.0.0.10
dockerBridgeCidr: 172.17.0.1/16
ipFamilies:
- IPv4
kubeProxyConfig: null
loadBalancerProfile:
  allocatedOutboundPorts: null
  backendPoolType: nodeIPConfiguration
  effectiveOutboundIPs:
  - id: <snip>/providers/Microsoft.Network/publicIPAddresses/562e450d-cb85-471c-ab36-2973d2add871
    resourceGroup: MC_richeneyaks_richeneyaks_uksouth
  enableMultipleStandardLoadBalancers: null
  idleTimeoutInMinutes: null
  managedOutboundIPs:
    count: 1
    countIpv6: null
  outboundIPs: null
  outboundIpPrefixes: null
loadBalancerSku: Standard
natGatewayProfile: null
networkDataplane: azure
networkMode: null
networkPlugin: azure
networkPluginMode: overlay
networkPolicy: calico
outboundType: loadBalancer
podCidr: 10.244.0.0/16
podCidrs:
- 10.244.0.0/16
serviceCidr: 10.0.0.0/16
serviceCidrs:
- 10.0.0.0/16
```

Merge credentials into ~/.kube/config.

```bash
az aks get-credentials --name $AKS_CLUSTER
```

Check

```bash
kubectl get nodes
```

```text
NAME                                STATUS     ROLES   AGE     VERSION
aks-nodepool1-24813231-vmss000000   NotReady   agent   2d20h   v1.25.6
aks-nodepool1-24813231-vmss000001   NotReady   agent   2d20h   v1.25.6
aks-nodepool1-24813231-vmss000002   NotReady   agent   2d20h   v1.25.6
```

Add namespaces:

```bash
kubectl create namespace alpha
kubectl create namespace beta
kubectl get ns
```

> If you have installed kubectx and kubens then `kubectl ns` will allow you to interactively switch.

## Create customer resources

```bash
export AKS_OIDC_ISSUER="$(az aks show --name "${AKS_CLUSTER}" --query "oidcIssuerProfile.issuerUrl" -otsv)"

for customer in alpha beta
do
  (
    # The parentheses means each loop iteration runs in a subshell.
    # Any variables (or env var changes) will be discarded at the end of the loop.
    # So changing AZURE_DEFAULTS_GROUP only defaults --resource-group
    # to the customer one for the duration of the loop iteration.

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

    kubectl apply -f - <<EOF1
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: "${client_id}"
  name: "${identity}"
  namespace: "${customer}"
EOF1

    kubectl apply -f - <<EOF2
apiVersion: v1
kind: Pod
metadata:
  name: quick-start
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
  )
done
```

## Secrets

Install [Helm](https://helm.sh/docs/intro/install/) if not already installed

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Install the [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/getting-started/installation.html)

```bash
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system
```

OK, drop the CNI Overlay and restart??