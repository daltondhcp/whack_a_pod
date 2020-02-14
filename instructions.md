# Instructions to deploy whack-a-pod on AKS 

These instructions assume that you have basic knowledge of Azure and Kubernetes. :smiley: 

### ***Prerequisites - tooling***

Unless you already have helm, kubectl az cli and git on your machine, I recommend you to use e Azure Cloud Shell when following the instructions. Azure Cloud Shell runs in your browser, and comes pre-installed with all the tools you need to work with Kubernetes. You start the cloud shell in the [Azure Portal](https://portal.azure.com) on the "shell" button in the top left tool bar or by launching https://shell.azure.com if you prefer a full screen cli experience. 
(https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough#use-azure-cloud-shell)

You will be asked to create a storage account, accept that and give it a name. Then when the cloud shell starts, select bash.
 
---
## 1. Deploy the AKS cluster (if you don't have one already) 
In the example below, we create a resource group and a cluster with the simplest possible configuration. The creation usually takes between 5-10 minutes. 

```
+ # Set variables
clustername='wack-cluster'
resourcegroup='aksrg'
az group create --name $resourcegroup --location eastus2
az aks create --resource-group $resourcegroup --name $clustername --node-count 2 --generate-ssh-keys
```
## 2. Deploy the containers to Kubernetes
While there are more elegant ways to do this with package managers like HELM, we create the deployments with the command line in this example. It is up to you if you want to build the containers yourself or use the pre-built ones provided from my docker hub repo. 

```
az aks get-credentials -n $clustername -g $resourcegroup

az aks create --resource-group aksrg --name wack-cluster --node-count 2 --generate-ssh-keys
```
