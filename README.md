# Run the cheshire cat on Kubernetes

This is a simple example of how to run the [cheshire cat](http://github.com/cheshire-cat-ai/core) on Azure Kubernetes Service.

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/zioproto/kube-cheshire-cat)

Requires Service Principal Terraform variables to be set as [Codespace Secrets](https://github.com/settings/codespaces):

https://learn.microsoft.com/en-us/azure/developer/terraform/authenticate-to-azure?tabs=bash#specify-service-principal-credentials-in-environment-variables

## Create an AKS cluster

Create an AKS cluster with the Azure Container Registry, build the image and push it to the registry.

```bash
#Create a resource group
az group create --name cheshire-cat --location eastus

REGISTRY=unique_name_for_the_registry

#Create a container registry
az acr create \
  --name ${REGISTRY} \
  --resource-group cheshire-cat \
  --sku Premium \
  --location eastus \
  --admin-enabled true

# Build the image
az acr login --name ${REGISTRY}
az acr build --registry ${REGISTRY} --image cheshire-cat-core:latest core/

#Create a cluster
az aks create \
 --location eastus \
 --name cheshire-cat \
 --enable-addons monitoring \
 --resource-group cheshire-cat \
 --network-plugin azure  \
 --node-vm-size Standard_DS3_v2 \
 --node-count 2 \
 --auto-upgrade-channel rapid \
 --node-os-upgrade-channel  NodeImage \
 --attach-acr ${REGISTRY}

# Get credentials
 az aks get-credentials --resource-group cheshire-cat --name cheshire-cat --overwrite-existing
```

## Install Gateway API

Install the Gateway API CRDs

```bash
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd/experimental?ref=v0.7.1" | kubectl apply -f -; }
```

## Install Istio

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
helm upgrade istio-base istio/base \
    --install \
    --create-namespace \
    --wait \
    --namespace istio-system \
    --version 1.17.3
helm upgrade istiod istio/istiod \
    --install \
    --create-namespace \
    --wait \
    --namespace istio-system \
    --version 1.17.3
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
helm upgrade --install --wait cheshire-cat qdrant/qdrant
```

## Install the Cheshire Cat

Install the core and the admin Deployment and ClusterIP Services.
Customize the following variables:

* UNIQUE_DNS_PREFIX: a unique prefix for the DNS name of the services

```bash
HUB=${REGISTRY}.azurecr.io
UNIQUE_DNS_PREFIX=cheshire-cat
API_KEY=$(openssl rand -hex 16 | base64)
cat kubernetes/cheshire-cat.yaml | \
sed -e "s/HUB/${HUB}/" |
sed -e "s/UNIQUE_DNS_PREFIX/${UNIQUE_DNS_PREFIX}/" |
sed -e "s/SECRET_VALUE/${API_KEY}/" |
kubectl apply -f -
```

## Install the gateway

Install the Istio Gateway and HTTPRoute. It will use cert-manager to generate a TLS certificate for the DNS name of the gateway.

Customize the following variables:
* EMAIL: the email address to use for the TLS certificate. Change with a real email or the certificate will not be generated.
* UNIQUE_DNS_PREFIX: a unique prefix for the DNS name of the services

```bash
EMAIL=email@example.com
UNIQUE_DNS_PREFIX=cheshire-cat
cat kubernetes/gateway.yaml | \
sed -e "s/EMAIL/${EMAIL}/" |
sed -e "s/UNIQUE_DNS_PREFIX/${UNIQUE_DNS_PREFIX}/" |
sed -e "s/SECRET_VALUE/${API_KEY}/" |
kubectl apply -f -
```

## Configure basic authentication for the admin interface:

Edit the file `kubernetes/authentication.yaml` to set the username and password for the admin interface.

Then apply the configuration:

```bash
 kubectl apply -f kubernetes/authentication.yaml
 ```

You can now access:
* the core API at https://${UNIQUE_DNS_PREFIX}.eastus.cloudapp.azure.com/
* the admin interface at https://${UNIQUE_DNS_PREFIX}.eastus.cloudapp.azure.com/admin/

Change in the URL the region and the DNS name to match your deployment.

# Large Language Models

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

CORE_URL="https://${UNIQUE_DNS_PREFIX}.eastus.cloudapp.azure.com"

curl -X 'PUT' \
  ''"$CORE_URL"'/settings/llm/LLMAzureOpenAIConfig' \
  -H "access_token: $API_KEY"
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
```
## Optional: Add a NodePools with GPUs to run local models

Create a nodepool with a VM SKU `Standard_NC48ads_A100_v4` that has 2 A100 GPUs.

```
az aks nodepool add \
    --resource-group cheshire-cat \
    --cluster-name  cheshire-cat \
    --name gpunp \
    --node-count 1 \
    --node-vm-size Standard_NC48ads_A100_v4 \
    --node-taints sku=gpu:NoSchedule \
    --aks-custom-headers UseGPUDedicatedVHD=true

```

Create a Pod running the [HuggingFace container](https://github.com/huggingface/text-generation-inference) `text-generation-inference`.
This container downloads and runs models from the HuggingFace Hub, and exposes a REST API to generate text.

```
kubectl apply -f kubernetes/falcon-40b-instruct.yaml
```

Configure the cheshire-cat core to use the HuggingFace container:

```
CORE_URL="https://${UNIQUE_DNS_PREFIX}.eastus.cloudapp.azure.com"
curl -X 'PUT' \
  ''"$CORE_URL"'/settings/llm/LLMHuggingFaceTextGenInferenceConfig' \
  -H "access_token: $API_KEY" \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
"inference_server_url": "http://text-generation-inference"
}'
```

The SKU `Standard_NC48ads_A100_v4` is more expensive than a regular VM without GPU.
You can delete the node pool when you don't need it anymore.

Remove the pool:
```
az aks nodepool delete \
    --resource-group cheshire-cat \
    --cluster-name  cheshire-cat \
    --name gpunp

```
