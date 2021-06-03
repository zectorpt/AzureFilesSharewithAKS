# AzureFilesSharewithAKS
# Change these four parameters as needed for your own environment
AKS_PERS_STORAGE_ACCOUNT_NAME=mystorageaccount$RANDOM  
AKS_PERS_RESOURCE_GROUP=myAKSShare  
AKS_PERS_LOCATION=eastus  
AKS_PERS_SHARE_NAME=aksshare  

# Create a resource group
az group create --name $AKS_PERS_RESOURCE_GROUP --location $AKS_PERS_LOCATION

# Create a storage account
az storage account create -n $AKS_PERS_STORAGE_ACCOUNT_NAME -g $AKS_PERS_RESOURCE_GROUP -l $AKS_PERS_LOCATION --sku Standard_LRS

# Export the connection string as an environment variable, this is used when creating the Azure file share
export AZURE_STORAGE_CONNECTION_STRING=$(az storage account show-connection-string -n $AKS_PERS_STORAGE_ACCOUNT_NAME -g $AKS_PERS_RESOURCE_GROUP -o tsv)

# Create the file share
az storage share create -n $AKS_PERS_SHARE_NAME --connection-string $AZURE_STORAGE_CONNECTION_STRING

# Get storage account key
STORAGE_KEY=$(az storage account keys list --resource-group $AKS_PERS_RESOURCE_GROUP --account-name $AKS_PERS_STORAGE_ACCOUNT_NAME --query "[0].value" -o tsv)

# Echo storage account name and key
echo Storage account name: $AKS_PERS_STORAGE_ACCOUNT_NAME  
echo Storage account key: $STORAGE_KEY

# Create a Kubernetes secret
kubectl create secret generic azure-secret --from-literal=azurestorageaccountname=$AKS_PERS_STORAGE_ACCOUNT_NAME --from-literal=azurestorageaccountkey=$STORAGE_KEY

# Mount file share as an inline volume
## Create a yaml azure-files-pod.yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: mypod  
spec:  
  containers:  
  - image: mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine  
    name: mypod  
    resources:  
      requests:  
        cpu: 100m  
        memory: 128Mi  
      limits:  
        cpu: 250m  
        memory: 256Mi  
    volumeMounts:  
      - name: azure  
        mountPath: /mnt/azure  
  volumes:  
  - name: azure  
    azureFile:  
      secretName: azure-secret  
      shareName: aksshare  
      readOnly: false  

