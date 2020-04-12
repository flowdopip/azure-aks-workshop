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
--service-principal 95200784-9925-41e6-8a78-229c67426b05 \
--client-secret O?k_Ahiq8QyU@ucg=an0D23s609vMg8X \
--node-count 2
--node-vm-size standard_b1ms

az aks get-credentials \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_CLUSTER_NAME

kubectl get nodes

kubectx

# create cluster role binding to the dashboard
kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard

az aks browse --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME

# get token
kubectl -n kube-system get secrets
kubectl -n kube-system describe secret kubernetes-dashboard-token-2zcqj| awk '$1=="token:"{print $2}'

kubectl -n kube-system describe secret $(kubectl get secrets -n kube-system -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}' | grep admin-user-token) | awk '$1=="token:"{print $2}'
eyJhbGciOiJSUzI1NiIsImtpZCI6InVoaWlNVklhandaTnRqYk90N2NRbEVKS0N5ZkV5VU5SbjdUVDN3MFNyVkUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJyZXNvdXJjZXF1b3RhLWNvbnRyb2xsZXItdG9rZW4tYzR3N2wiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoicmVzb3VyY2VxdW90YS1jb250cm9sbGVyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNzJmYzY5ZDItMjM0MC00ODA5LTlkOGMtYmQwZmEzNmJkNzE0Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOnJlc291cmNlcXVvdGEtY29udHJvbGxlciJ9.lBSuU5O7tGYNmkQ-zzkBrypSlcRdx0G7JWeJR-FBKHI_Q7YqPNyKD1StCBTOiCpy40OZHyaPQ7-HQ5YEGVL4aCjdYMxMsCFr_ERN1U7N4YaQQuQ8cQYegV5O4DdSbQ3vWI7krwtwhI4qGmyc5YiddLJhU4laQYQX7bM0S0ISYz5uZYRS1tTMdteQNyMX0XSVmOvSpgiRiI2JZkWqmN51pgEbsVMuiLYbuiTdZOJnJmZ9-IGTvbed0jX8163pu0UOjwo30TPxdMVJqALovEdx2oWW4vMNvbXKvMZC0hCBO1wPjTYawuWgSjjOdqeHlMMW-cs7_8rEk2L2zzpYJWVVr9GVdS1rJb_dI2DeeO2jTupcweD0lgUnqW3yI3DSx7dXKQtieyo68dlSIYMVxbA0QCjfMoeAjBO1I_CBxYt391RWb0VRxA7il47F4SuSUnp_BstV4PxVtzGiec5OdcAv2uebIcI4_ZiU-3rSYnzh0vZU5aIdKpKgkMCnfuZRLa-QJUWaXT5ZF9lJKyafotOpjhOGKFOsBZTNu3jX47dYbouubFhYNiOqgyqEl0bJ8Qza4mvfhEL5Nv-zmweWvXrh1lLtAjJAq8-YL47A25AEiKi7jURCKabdLcJPZvppAmm_SY3QLHNWiUoF7uu_FEglI2CxUL-sM-haWTARgjGEAFY

```

## Applications
```shel

kubectl create namespace ratingsapp

```

## Container Registry
```shell
ACR_NAME=azureworkshopregistry

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
helm repo add stable https://kubernetes-charts.storage.googleapis.com/

helm search repo stable

helm install ratings stable/mongodb \
    --namespace ratingsapp \
    --set mongodbUsername=mongoadmin,mongodbPassword=mongopwd,mongodbDatabase=ratingsdb

kubectl create secret generic mongosecret \
    --namespace ratingsapp \
    --from-literal=MONGOCONNECTION="mongodb://mongoadmin:mongopwd@ratings-mongodb.ratingsapp.svc.cluster.local:27017/ratingsdb"

kubectl describe secret mongosecret --namespace ratingsapp

kubectl apply \
    --namespace ratingsapp \
    -f ratings-api-deployment.yaml

kubectl get pods \
    --namespace ratingsapp \
    -l app=ratings-api -w

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
```