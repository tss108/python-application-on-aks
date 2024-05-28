# Python Application on Azure Kubernetes Service (AKS)
This guide provides instructions on how to create an AKS cluster, build a Docker image for a simple Python application, push the image to an Azure Container Registry (ACR), and deploy the application to the AKS cluster.

## Prerequisites
- **Azure CLI:** Installed and configured on your local machine. This is necessary for managing Azure resources.
- **Docker:** Installed on your local machine. This is required to build and push container images.
- **Azure Account:** You should have an Azure account with appropriate permissions to create and manage resources.
- **Knowledge of Kubernetes:** Understanding of Kubernetes concepts and experience with kubectl commands.
- **Knowledge of Docker:** Familiarity with Docker, including building and pushing Docker images.
- **Azure Container Registry (ACR):** Understanding of how to create and use Azure Container Registry to store Docker images.
- **Azure Container Instances (ACI):** Basic knowledge of how to deploy and manage container instances on Azure.


## Project Directory Layout
```bash
.
├── Dockerfile
├── greetings-service-ec.yaml
├── greetings-service-ic.yaml
├── k8s-deployment.yaml
├── main.py
├── README.md
├── redis-service.yaml
├── redis.yaml
└── requirements.txt
```

## File Descriptions

- **Dockerfile**
  - This file contains the instructions for building the Docker image of the Greetings application. It specifies the base image, sets up the working directory, installs dependencies, copies application files, and defines the entry point for the container.

- **greetings-service-ec.yaml**
  - Kubernetes manifest for the Greetings application's external service. This file defines the configuration for exposing the application to external traffic using a Kubernetes LoadBalancer service.

- **greetings-service-ic.yaml**
  - Kubernetes manifest for the Greetings application's internal service. This file configures the service to be accessible only within the cluster using a ClusterIP service type.

- **k8s-deployment.yaml**
  - Kubernetes deployment manifest for the Greetings application. It describes the deployment of the application's pods, specifying the number of replicas, container image, environment variables, and node selector to assign pods to the appropriate node pool.

- **main.py**
  - The main Python script for the Greetings application. This script contains the code for the web application, handling HTTP requests, and returning greeting messages.

- **redis-service.yaml**
  - Kubernetes service manifest for the Redis instance. This file defines the configuration for exposing the Redis service within the cluster, enabling the Greetings application to connect to the Redis back-end.

- **redis.yaml**
  - Kubernetes deployment manifest for the Redis instance. It describes the deployment configuration for Redis, including resource requests, node selector, and other necessary parameters to ensure Redis runs on the specified node pool.

- **requirements.txt**
  - A file specifying the Python dependencies required by the Greetings application. It is used by the Dockerfile to install the necessary packages inside the container.

## How To Use:

### 1. Log in to Azure
```bash
  az login
```
### 2. Create a Resource Group
```bash
  az group create --name <RESOURCE_GROUP_NAME> --location <LOCATION>
```
*Example:*
```bash
  az group create --name my-aks-rg --location eastus <LOCATION>
```
### 3. Create an AKS Cluster
```bash
az aks create --resource-group <RESOURCE_GROUP_NAME> --name <AKS_CLUSTER_NAME> --enable-managed-identity --node-count <NODE_COUNT> --generate-ssh-keys
```
*Example:*
```bash
  az aks create --resource-group my-aks-rg --name my-aks-cluster --enable-managed-identity --node-count 1 --generate-ssh-keys
```

### 4. Add Node Pools
#### Frontend Node Pool
```bash
az aks nodepool add --resource-group <RESOURCE_GROUP_NAME> --cluster-name <AKS_CLUSTER_NAME> --name <FRONTEND_NODEPOOL_NAME> --node-count <NODE_COUNT> --labels app=<FRONTEND_APP_LABEL>
```
*Example:*
```bash
az aks nodepool add --resource-group my-aks-rg --cluster-name my-aks-cluster --name frontendpool --node-count 1 --labels app=greetings-app
```

#### Backend Node Pool
```bash
az aks nodepool add --resource-group <RESOURCE_GROUP_NAME> --cluster-name <AKS_CLUSTER_NAME> --name <BACKEND_NODEPOOL_NAME> --node-count <NODE_COUNT> --labels app=<BACKEND_APP_LABEL>
```
*Example:*
```bash
az aks nodepool add --resource-group my-aks-rg --cluster-name my-aks-cluster --name backendpool --node-count 1 --labels app=redis-app
```

### 5. Create an Azure Container Registry
```bash
az acr create --resource-group <RESOURCE_GROUP_NAME> --name <ACR_NAME> --sku Basic
```
*Example:*
```bash
az acr create --resource-group my-aks-rg --name myaksacr --sku Basic
```

### 6. Build and Push Docker Image
#### Build Docker Image

```bash
sudo docker build -t <DOCKER_IMAGE_NAME> .
```
*Example:*
```bash
sudo docker build -t greetings-app .
```

#### Get ACR Login Server

```bash
az acr show --name <ACR_NAME> --resource-group <RESOURCE_GROUP_NAME> --query loginServer --output tsv
```
*Example:*
```bash
az acr show --name myaksacr --resource-group my-aks-rg --query loginServer --output tsv
```

#### Tag Docker Image
```bash
sudo docker tag <DOCKER_IMAGE_NAME> <ACR_LOGIN_SERVER>/<DOCKER_IMAGE_NAME>:latest
```
*Example:*
```bash
sudo docker tag greetings-app myaksacr.azurecr.io/greetings-app:latest
```

#### Log in to ACR
```bash
sudo docker login <ACR_LOGIN_SERVER>
```
*Example:*
```bash
sudo docker login myaksacr.azurecr.iot
```

#### Push Docker Image to ACR
```bash
sudo docker push <ACR_LOGIN_SERVER>/<DOCKER_IMAGE_NAME>:latest
```
*Example:*
```bash
sudo docker push myaksacr.azurecr.io/greetings-app:latest
```

### 7. Set Azure Subscription
```bash
az account set --subscription <SUBSCRIPTION_ID>
```
*Example:*
```bash
az account set --subscription 181b17b1-76d1-4338-b4ea-2gjhvh114dfe
```

### 8. Get AKS Credentials
```bash
az aks get-credentials --resource-group <RESOURCE_GROUP_NAME> --name <AKS_CLUSTER_NAME> --overwrite-existing
```
*Example:*
```bash
az aks get-credentials --resource-group my-aks-rg --name my-aks-cluster --overwrite-existing
```

### 9. Export KUBECONFIG Environment Variable
```bash
export KUBECONFIG=<KUBECONFIG_LOCATION>
```
*Example:*
```bash
export KUBECONFIG=/home/user/.kube/config
```

### 10. Create a ConfigMap
```bash
kubectl create configmap <CONFIGMAP_NAME> --from-literal=NAME=<NAME>
```
*Example:*
```bash
kubectl create configmap my-configmap --from-literal=NAME=MyAppName
```

### 11. Create a Docker Registry Secret
```bash
kubectl create secret docker-registry <SECRET_NAME> --docker-server=<ACR_LOGIN_SERVER> --docker-username=<ACR_NAME> --docker-password=<ACR_PASSWORD>
```
*Example:*
```bash
kubectl create secret docker-registry my-secret --docker-server=myaksacr.azurecr.io --docker-username=myaksacr --docker-password=<Password>
```

### 12. Apply Kubernetes Deployment Configuration
```bash
kubectl apply -f k8s-deployment.yaml
```

### 13. Apply Kubernetes Service Configurations
```bash
kubectl apply -f greetings-service-ic.yaml
kubectl apply -f greetings-service-ec.yaml
```

### 14. Get Kubernetes Deployments and Pods
```bash
kubectl get deployments
kubectl get pods
```

### 15. Apply Redis Configuration and Service
```bash
kubectl apply -f redis.yaml
kubectl apply -f redis-service.yaml
```

### 16. Get Kubernetes Services
```bash
kubectl get services
```

### 17. List Docker Images
```bash
sudo docker images
```

Make sure to replace the placeholders (`<RESOURCE_GROUP_NAME>`, `<AKS_CLUSTER_NAME>`, `<FRONTEND_NODEPOOL_NAME>`, `<BACKEND_NODEPOOL_NAME>`, `<ACR_NAME>`, `<DOCKER_IMAGE_NAME>`, `<CONFIGMAP_NAME>`, `<SECRET_NAME>`, etc.) with your actual values.