# ACR Cache

Additional detail for setting up a cache for Azure Container Registry (ACR).

This will be given a private endpoint in the future, but baby steps, right?

## Installing docker on the WSL2 CLI

You can use Docker Desktop for Windows, but I prefer to use the WSL2 CLI for quick testing.

1. Install Docker CLI

    ```bash
    sudo apt-get update && sudo apt-get install docker.io
    ```

1. Add your user to the docker group

    ```bash
    sudo usermod -aG docker $USER
    ```

1. Log out and back in again

1. Test docker

    ```bash
    docker run hello-world
    ```

## Setup

1. Set defaults for location and resource group using environment variables

    ```bash
    export AZURE_DEFAULTS_LOCATION=uksouth
    export AZURE_DEFAULTS_GROUP=richeneyaks
    export AZURE_DEFAULTS_ACR=richeneyacrcache
    ```

    No need to specify `--location`, `--resource-group` or `--registry` for the remainder of the commands.

    Note that the ACR name needs to be globally unique.

1. Create the resource group

    ```bash
    az group create --name $AZURE_DEFAULTS_GROUP
    ```

1. Create an ACR with the Azure CLI

    ```bash
    az acr create --name $AZURE_DEFAULTS_ACR --sku Basic
    ```

    [ACR SKU tiers](https://docs.microsoft.com/azure/container-registry/container-registry-skus)

1. Add the AcrPush role to the registry (optional)

    ```bash
    az role assignment create --role AcrPush \
    --assignee $(az ad signed-in-user show --query id -otsv) \
    --scope $(az acr show --name $AZURE_DEFAULTS_ACR --query id -otsv)
    ```

    The Owner and Contributor roles can push and pull images automatically. Example AcrPush included for completeness. Use AcrPull if only pulling images.

    [Container Registry Roles](https://learn.microsoft.com/azure/container-registry/container-registry-roles?tabs=azure-cli) for service principal CICD alternatives.

## Config

1. Create a cache rule for the registry

    ```bash
    az acr cache create --name myCacheRule \
      --source-repo mcr.microsoft.com/azuredocs/aks-helloworld \
      --target-repo aks-helloworld
    ```

1. Show the cache rule

    ```bash
    az acr cache show --name myCacheRule
    ```

## Pull

1. Authenticate to the registry

   Manual authentication shown.

   ```bash
    az acr login
    ```

    [Container Registry Authentication](https://learn.microsoft.com/azure/container-registry/container-registry-authentication?tabs=azure-cli)

1. List the repos

    ```bash
    az acr repository list -otsv
    ```

1. List the most recent tag for the repo

    ```bash
    az acr repository show-tags --repository aks-helloworld \
      --orderby time_desc --top 1 -otsv
    ```

1. Pull the image

    ```bash
    docker pull $AZURE_DEFAULTS_ACR.azurecr.io/aks-helloworld:v1
    ```
