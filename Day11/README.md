# Day 11/16 - Azure DevOps CICD Pipeline on Azure Kubernetes Services 🛳️ 🐳

## Check out the video below for Day11 👇

[![Day10/16 - Azure DevOps CICD Pipeline on Azure Container Instances](https://img.youtube.com/vi/mv8qJzyZggE/sddefault.jpg)](https://youtu.be/mv8qJzyZggE)


## Kubernetes Architecture

<img width="820" alt="image" src="https://github.com/piyushsachdeva/AzureDevOps-Zero-to-Hero/assets/40286378/e15fe0dc-afc0-4aec-b69f-b7e296df0b07">

## Kubernetes components

### Kube APiServer

<img width="746" alt="image" src="https://github.com/piyushsachdeva/AzureDevOps-Zero-to-Hero/assets/40286378/808131a2-e8b5-4b89-af47-5f1040c64544">

### Kube Scheduler

<img width="745" alt="image" src="https://github.com/piyushsachdeva/AzureDevOps-Zero-to-Hero/assets/40286378/7e394ee3-37c9-4ade-82b0-95da0b0318ef">

### Kube Controller Manager

<img width="853" alt="image" src="https://github.com/piyushsachdeva/AzureDevOps-Zero-to-Hero/assets/40286378/4dffd765-559c-4839-8152-7f15ab753adf">

### ETCD

<img width="817" alt="image" src="https://github.com/piyushsachdeva/AzureDevOps-Zero-to-Hero/assets/40286378/f50887ea-28d0-40b3-968d-1e923f2177a4">


## Pre-requisites

### Infrastructure provisioning

**Using the below script, you can provision infra for this demo**

The following resources will be provisioned:

* A Resource Group
* An AKS Kubernetes Cluster
* An Image container registry
* A SQL Server
* A SQL Database

before running this script do dos2unix <scriptname> if edited the script in windows.

``` bash

#!/bin/bash

# Variables
LOCATION="canadacentral"
RG_NAME="day11-demo-rg"
AKS_NAME="day11-demo-cluster"
ACR_NAME="day11demoacr$RANDOM"  # Unique name to avoid conflicts
SQLSERVER_NAME="day11demosql$RANDOM"  # Unique name (SQL server names must be globally unique)
DB_NAME="mhcdb"
SQL_ADMIN_USER="sqladmin"
SQL_ADMIN_PASSWORD="P@ssw0rd1234!"  # Strong password
NODE_SIZE="Standard_B2s"
NODE_COUNT=1

# Select your subscription (optional)
# az account set --subscription "<your-subscription-id>"

# Register required resource providers
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.Insights
az provider register --namespace Microsoft.Sql

# Create resource group
echo "Creating Resource Group..."
az group create --name $RG_NAME --location $LOCATION

# Create ACR
echo "Creating Azure Container Registry..."
az acr create --resource-group $RG_NAME --name $ACR_NAME --sku Basic --location $LOCATION
if [ $? -ne 0 ]; then
    echo "❌ Failed to create ACR. Exiting."
    exit 1
fi

# Create AKS cluster
echo "Creating AKS Cluster..."
az aks create \
  --resource-group $RG_NAME \
  --name $AKS_NAME \
  --node-count $NODE_COUNT \
  --node-vm-size $NODE_SIZE \
  --enable-managed-identity \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --location $LOCATION \
  --attach-acr $ACR_NAME
if [ $? -ne 0 ]; then
    echo "❌ Failed to create AKS Cluster. Exiting."
    exit 1
fi

# Get AKS credentials
az aks get-credentials --resource-group $RG_NAME --name $AKS_NAME

# Create SQL Server
echo "Creating SQL Server..."
az sql server create \
  --name $SQLSERVER_NAME \
  --resource-group $RG_NAME \
  --location $LOCATION \
  --admin-user $SQL_ADMIN_USER \
  --admin-password $SQL_ADMIN_PASSWORD
if [ $? -ne 0 ]; then
    echo "❌ Failed to create SQL Server. Exiting."
    exit 1
fi

# Create SQL Database
echo "Creating SQL Database..."
az sql db create \
  --resource-group $RG_NAME \
  --server $SQLSERVER_NAME \
  --name $DB_NAME \
  --service-objective S0
if [ $? -ne 0 ]; then
    echo "❌ Failed to create SQL Database. Exiting."
    exit 1
fi

# Summary
echo "✅ Deployment Successful!"
echo "AKS: $AKS_NAME"
echo "ACR: $ACR_NAME"
echo "SQL Server: $SQLSERVER_NAME"
echo "SQL DB: $DB_NAME"



```

## Change the Firewall settings of the SQL server

![image](https://github.com/piyushsachdeva/AzureDevOps-Zero-to-Hero/assets/40286378/d421dd8b-1a85-447a-ad2d-f0ddb859953d)


## Setup Azure DevOps Project

### Pre-requisites

Make sure the below Azure DevOps extensions are installed and enabled in your organization
- Replace Token
- Kubernetes extension

Once the infra is ready, go to dev.azure.com --> Project --> repos 
and import the below git repo, which has the source code and pipeline code

https://github.com/piyushsachdeva/MyHealthClinic-AKS

### Build and Release Pipeline

- You can create your pipeline by following along the video or editing the existing pipeline. The below details need to be updated in the pipeline:
   - Azure Service connection
   - Token pattern
   - Pipeline variables
   - The Kubectl version should be the latest in the release pipeline
   - Secrets should be updated in the deployment step
   - ACR details in the pipeline should be updated
 
#### Steps in the pipeline

<img width="805" alt="image" src="https://github.com/piyushsachdeva/AzureDevOps-Zero-to-Hero/assets/40286378/64907bef-acbf-41f1-b3de-63b46681db3d">

 [Image source](https://azuredevopslabs.com/labs/vstsextend/kubernetes/)

## Destroy the resources at the end of the demo

```
#!/bin/bash


# Set environment variables
REGION="westus"
RGP="day11-demo-rg"
CLUSTER_NAME="day11-demo-cluster"
ACR_NAME="day11demoacr"
SQLSERVER="day11-demo-sqlserver"
DB="mhcdb"

# Function to handle errors
handle_error() {
    echo "Error: $1"
    exit 1
}

# Function to check if the resource exists
resource_exists() {
    az resource show --ids $1 &> /dev/null
}

# Delete Azure Kubernetes Service (AKS)
if resource_exists $(az aks show --resource-group $RGP --name $CLUSTER_NAME --query id --output tsv); then
    az aks delete --resource-group $RGP --name $CLUSTER_NAME || handle_error "Failed to delete AKS."
else
    echo "AKS not found. Skipping deletion."
fi

# Delete Azure Container Registry (ACR)
if resource_exists $(az acr show --name $ACR_NAME --resource-group $RGP --query id --output tsv); then
    az acr delete --name $ACR_NAME --resource-group $RGP || handle_error "Failed to delete ACR."
else
    echo "ACR not found. Skipping deletion."
fi

# Delete SQL Database
if resource_exists $(az sql db show --resource-group $RGP --server $SQLSERVER --name $DB --query id --output tsv); then
    az sql db delete --resource-group $RGP --server $SQLSERVER --name $DB || handle_error "Failed to delete SQL Database."
else
    echo "SQL Database not found. Skipping deletion."
fi

# Delete SQL Server
if resource_exists $(az sql server show --resource-group $RGP --name $SQLSERVER --query id --output tsv); then
    az sql server delete --resource-group $RGP --name $SQLSERVER || handle_error "Failed to delete SQL Server."
else
    echo "SQL Server not found. Skipping deletion."
fi

# Delete Resource Group
if resource_exists $(az group show --name $RGP --query id --output tsv); then
    az group delete --name $RGP || handle_error "Failed to delete Resource Group."
else
    echo "Resource Group not found. Skipping deletion."
fi

echo "Resources successfully deleted."

```

