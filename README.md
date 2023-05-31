# Run the cheshire cat on Kubernetes

This is a simple example of how to run the [cheshire cat](http://github.com/pieroit/cheshire-cat) on Azure Kubernetes Service.

**WARNING**: this is a proof of concept, not a production ready deployment. The admin container running the Node.js web interface does not implement any authentication or authorization to connect to the core containers. The core container is exposed to the internet without any authentication or authorization, and it is possible to leak your OpenAI keys to the world. Use at your own risk.

* https://github.com/pieroit/cheshire-cat/issues/198
* PR https://github.com/zioproto/kube-cheshire-cat/pull/1

## Docker images

The necessary Docker images are available at:
* zioproto/cheshire-cat-core [Dockerfile](core/Dockerfile)

## Create an AKS cluster

Create an AKS cluster with the Azure Service Mesh addon, which is required for the Gateway API to work.

```bash
# Update the aks-preview extension
az extension update --name aks-preview

# Register the preview feature AzureServiceMesh
az feature register --namespace "Microsoft.ContainerService" --name "AzureServiceMeshPreview"

# Wait for RegistrationState to be "Registered"
az feature show --namespace "Microsoft.ContainerService" --name "AzureServiceMeshPreview"

# Register the provider again
az provider register -n Microsoft.ContainerService

#Create a resource group
az group create --name cheshire-cat --location eastus

#Create a cluster
az aks create \
 --location eastus \
 --name cheshire-cat \
 --enable-addons monitoring \
 --resource-group cheshire-cat \
 --network-plugin azure  \
 --kubernetes-version 1.25.6  \
 --node-vm-size Standard_DS3_v2 \
 --node-count 2 \
 --auto-upgrade-channel rapid \
 --node-os-upgrade-channel  NodeImage \
 --enable-asm

# Get credentials
 az aks get-credentials --resource-group cheshire-cat --name cheshire-cat --overwrite-existing
```

## Install Gateway API

Install the Gateway API CRDs

```bash
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd/experimental?ref=v0.6.1" | kubectl apply -f -; }
```

## Install cert-manager

Install the cert-manager helm chart with the Gateway API support enabled.

More docs here: https://cert-manager.io/docs/usage/gateway/

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade cert-manager jetstack/cert-manager \
    --install \
    --create-namespace \
    --wait \
    --namespace cert-manager \
    --set installCRDs=true \
    --set "extraArgs={--feature-gates=ExperimentalGatewayAPISupport=true}"
```

## Install qdrant

```bash
helm repo add qdrant https://qdrant.github.io/qdrant-helm
helm repo update
helm install --wait cheshire-cat qdrant/qdrant
```

## Install the Cheshire Cat

Install the core and the admin Deployment and ClusterIP Services.
Customize the following variables:

* HUB: the Docker Hub account where the images are stored
* UNIQUE_DNS_PREFIX: a unique prefix for the DNS name of the services

```bash
HUB=zioproto
UNIQUE_DNS_PREFIX=cheshire-cat
cat kubernetes/cheshire-cat.yaml | \
sed -e "s/HUB/${HUB}/" |
sed -e "s/UNIQUE_DNS_PREFIX/${UNIQUE_DNS_PREFIX}/" |
kubectl apply -f -
```

## Install the gateway

Install the Istio Gateway and HTTPRoute. It will use cert-manager to generate a TLS certificate for the DNS name of the gateway.

Customize the following variables:
* EMAIL: the email address to use for the TLS certificate
* UNIQUE_DNS_PREFIX: a unique prefix for the DNS name of the services

```bash
EMAIL=email@example.com
UNIQUE_DNS_PREFIX=cheshire-cat
cat kubernetes/gateway.yaml | \
sed -e "s/EMAIL/${EMAIL}/" |
sed -e "s/UNIQUE_DNS_PREFIX/${UNIQUE_DNS_PREFIX}/" |
kubectl apply -f -
```
## Use Cheshire Cat with Azure OpenAI

You can use the Cheshire Cat with Azure OpenAI. Follow these steps:

Deploy an Azure OpenAI resource

```
az cognitiveservices account create \
-n cheshire-cat \
-g cheshire-cat \
-l eastus \
--kind OpenAI \
--sku s0
```

Deploy the models for completion and for embeddings:

```
az cognitiveservices account deployment create \
   -g cheshire-cat\
   -n cheshire-cat \
   --deployment-name gpt-35-turbo \
   --model-name gpt-35-turbo \
   --model-version "0301"  \
   --model-format OpenAI \
   --scale-settings-scale-type "Standard"

az cognitiveservices account deployment create \
   -g cheshire-cat\
   -n cheshire-cat \
   --deployment-name text-embedding-ada-002 \
   --model-name text-embedding-ada-002 \
   --model-version "2"  \
   --model-format OpenAI \
   --scale-settings-scale-type "Standard"
```

Configure to core container to use the Azure OpenAI models you just deployed,
retrieve the key and the api endpoint with Azure CLI and push them to the
core with an API request:

```
#!/bin/bash
ENDPOINT=$(az cognitiveservices account show -o json \
--name cheshire-cat \
--resource-group cheshire-cat \
| jq -r .properties.endpoint)


KEY=$(az cognitiveservices account keys list -o json \
--name cheshire-cat \
--resource-group cheshire-cat \
| jq -r .key1)

CORE_URL="https://${UNIQUE_DNS_PREFIX}-core.eastus.cloudapp.azure.com"

curl -X 'PUT' \
  ''"$CORE_URL"':1865/settings/llm/LLMAzureOpenAIConfig' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "api_type": "azure",
  "api_version": "2022-12-01",
  "deployment_name": "gpt-35-turbo",
  "model_name": "gpt-35-turbo",
  "openai_api_base": "'"$ENDPOINT"'",
  "openai_api_key": "'"$KEY"'"
}'

