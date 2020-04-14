# azure-aks-workshop
Azure AKS Workshop: https://docs.microsoft.com/en-us/learn/modules/aks-workshop/

## Introduction
```shell

REGION_NAME=eastus
RESOURCE_GROUP=aks-workshop
SUBNET_NAME=subnet-aks-workshop
VNET_NAME=vnet-aks-workshop

# login to Azure
az login

# create a resource group for the workshop
az group create --name aks-workshop --location eastus
az group list --output table

# create a vnet and a subnet for the workshop
az network vnet create --resource-group aks-workshop --location eastus --name vnet-aks-workshop --address-prefixes 10.0.0.0/8 --subnet-name subnet-aks-workshop --subnet-prefix 10.240.0.0/16

az network vnet list --output table

# get the subnetid
SUBNET_ID=$(az network vnet subnet show --resource-group aks-workshop --vnet-name vnet-aks-workshop --name subnet-aks-workshop --query id -o tsv)

```

## AKS
```shell

AKS_CLUSTER_NAME=aks-workshop-$RANDOM

# get the available aks versions
az aks get-versions \
    --location eastus \
    --query 'orchestrators[?!isPreview] | [-1].orchestratorVersion' \
    --output tsv

# create the aks
az aks create \
--resource-group $RESOURCE_GROUP \
--name $AKS_CLUSTER_NAME \
--vm-set-type VirtualMachineScaleSets \
--load-balancer-sku standard \
--location $REGION_NAME \
--kubernetes-version $VERSION \
--network-plugin azure \
--vnet-subnet-id $SUBNET_ID \
--service-cidr 10.2.0.0/24 \
--dns-service-ip 10.2.0.10 \
--docker-bridge-address 172.17.0.1/16 \
--generate-ssh-keys \
--service-principal $SERVICE_PRINCIPAL \
--client-secret $CLIENT_SECRET \
--node-count 2
--node-vm-size standard_b1ms

# get aks credentials
az aks get-credentials \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_CLUSTER_NAME

kubectl get nodes

kubectx

# create cluster role binding to the dashboard
kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard

# Open browser dashboard
az aks browse --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME

# get token
kubectl -n kube-system get secrets
kubectl -n kube-system describe secret kubernetes-dashboard-token-2zcqj| awk '$1=="token:"{print $2}'

# get cluster secrets
kubectl -n kube-system describe secret $(kubectl get secrets -n kube-system -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}' | grep admin-user-token) | awk '$1=="token:"{print $2}'

```

## Applications
```shel

# create namespace
kubectl create namespace ratingsapp

```

## Container Registry
```shell

ACR_NAME=azureworkshopregistry

# create a azure container registry
az acr create \
    --resource-group $RESOURCE_GROUP \
    --location $REGION_NAME \
    --name $ACR_NAME \
    --sku Standard
```

## ratings app
```shell

git clone https://github.com/MicrosoftDocs/mslearn-aks-workshop-ratings-api.git

cd mslearn-aks-workshop-ratings-api

# add rating-api to the acr
az acr build \
    --registry $ACR_NAME \
    --image ratings-api:v1 .

cd ..

git clone https://github.com/MicrosoftDocs/mslearn-aks-workshop-ratings-web.git

cd mslearn-aks-workshop-ratings-web

# add rating-web to the acr
az acr build \
    --registry $ACR_NAME \
    --image ratings-web:v1 .

# list act repositories
az acr repository list \
    --name $ACR_NAME \
    --output table

# authn for the aks and azure registry
az aks update \
    --name $AKS_CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --attach-acr $ACR_NAME

```

## setup mongo db
```shell

# add helm repo
helm repo add stable https://kubernetes-charts.storage.googleapis.com/

helm search repo stable

# install mongodb
helm install ratings stable/mongodb \
    --namespace ratingsapp \
    --set mongodbUsername=mongoadmin,mongodbPassword=mongopwd,mongodbDatabase=ratingsdb

# create secret for mongodb
kubectl create secret generic mongosecret \
    --namespace ratingsapp \
    --from-literal=MONGOCONNECTION="mongodb://mongoadmin:mongopwd@ratings-mongodb.ratingsapp.svc.cluster.local:27017/ratingsdb"

kubectl describe secret mongosecret --namespace ratingsapp

# deploy rating api
kubectl apply \
    --namespace ratingsapp \
    -f ratings-api-deployment.yaml

kubectl get pods \
    --namespace ratingsapp \
    -l app=ratings-api -w

# create rating api service
kubectl apply \
    --namespace ratingsapp \
    -f ratings-api-service.yaml

kubectl get service ratings-api --namespace ratingsapp

kubectl get endpoints ratings-api --namespace ratingsapp
```

## WEB
```shel 

kubectl apply \
--namespace ratingsapp \
-f ratings-web-deployment.yaml

kubectl get pods --namespace ratingsapp -l app=ratings-web -w

kubectl get deployment ratings-web --namespace ratingsapp

kubectl apply \
    --namespace ratingsapp \
    -f ratings-web-service.yaml

kubectl get service ratings-web --namespace ratingsapp -w
```

## Ingress Controller
```shell
kubectl create namespace ingress

helm install nginx-ingress stable/nginx-ingress \
    --namespace ingress \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux

kubectl get service nginx-ingress-controller --namespace ingress -w

kubectl delete service \
    --namespace ratingsapp \
    ratings-web

kubectl apply \
    --namespace ratingsapp \
    -f ratings-web-service.yaml

kubectl apply \
    --namespace ratingsapp \
    -f ratings-web-ingress.yaml
```

## Enable ssl/tls
```shel
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update

kubectl apply --validate=false -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.14/deploy/manifests/00-crds.yaml

helm install cert-manager \
    --namespace cert-manager \
    --version v0.14.0 \
    jetstack/cert-manager

kubectl get pods --namespace cert-manager

cd cert-manager
kubectl apply \
    --namespace ratingsapp \
    -f cluster-issuer.yaml

kubectl apply \
    --namespace ratingsapp \
    -f ratings-web-ingress.yaml

kubectl describe cert ratings-web-cert --namespace ratingsapp
```

## Monitoring 
```shell
WORKSPACE=aks-workshop-workspace-$RANDOM

az resource create --resource-type Microsoft.OperationalInsights/workspaces \
        --name $WORKSPACE \
        --resource-group $RESOURCE_GROUP \
        --location $REGION_NAME \
        --properties '{}' -o table

WORKSPACE_ID=$(az resource show --resource-type Microsoft.OperationalInsights/workspaces \
    --resource-group $RESOURCE_GROUP \
    --name $WORKSPACE \
    --query "id" -o tsv)

az aks enable-addons \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_CLUSTER_NAME \
    --addons monitoring \
    --workspace-resource-id $WORKSPACE_ID

cd monitor

# setup Role and Cluster Role binding
kubectl apply \
    -f logreader-rbac.yaml
```

## Horizontal POD Autoscaler

