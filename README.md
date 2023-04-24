# Run the cheshire cat on Kubernetes

This is a simple example of how to run the [cheshire cat](http://github.com/pieroit/cheshire-cat) on Azure Kubernetes Service.

## Docker images

The necessary Docker images are available at:
* zioproto/cheshire-cat-admin [Dockerfile](admin/Dockerfile)
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
 az aks get-credentials --resource-group cheshire-cat --name cheshire-cat
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
helm install cheshire-cat qdrant/qdrant
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


