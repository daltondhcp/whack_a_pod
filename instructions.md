# Instructions to deploy whack-a-pod on AKS 

These instructions assume that you have basic knowledge of Azure and Kubernetes. :smiley: 

### ***Prerequisites - tooling***

Unless you already have helm, kubectl az cli and git on your machine, I recommend you to use Azure Cloud Shell when following the instructions. Azure Cloud Shell runs in your browser, and comes pre-installed with all the tools you need to work with Kubernetes. You start the cloud shell in the [Azure Portal](https://portal.azure.com) on the "shell" button in the top left tool bar or by launching https://shell.azure.com if you prefer a full screen cli experience. 
(https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough#use-azure-cloud-shell)

You will be asked to create a storage account, accept that and give it a name. Then when the cloud shell starts, **select bash**.
 
---
## 1. Deploy the AKS cluster (if you don't have one already) 
In the example below, we create a resource group and a cluster with the simplest possible configuration. The creation usually takes between 5-10 minutes. 

```
# Set variables
clustername='wack-cluster'
resourcegroup='aksrg'
dockerrepo='daltondhcp'
#Create resource group and cluster
az group create --name $resourcegroup --location eastus2
az aks create --resource-group $resourcegroup --name $clustername --node-count 2 --generate-ssh-keys
```
## 2. Deploy the containers to Kubernetes
While there are more elegant ways to do this with package managers like HELM, we create the deployments with the command line in this example. It is up to you if you want to build the containers yourself or use the pre-built ones provided from my docker hub repo. 


#### Get credentials for kubernetes cluster
```
az aks get-credentials -n $clustername -g $resourcegroup
```
#### Deploy and expose admin container
```
kubectl run admin-deployment --image=$dockerrepo/whackapod-admin --replicas=1 --port=8080 --labels=app=admin --env="APIIMAGE=$dockerrepo/whackapod-api"
kubectl expose deployment admin-deployment --name=admin --target-port=8080  --type=NodePort --labels="app=admin"
kubectl create serviceaccount wap-admin	
kubectl create clusterrolebinding wap-admin --clusterrole=cluster-admin --serviceaccount=default:wap-admin
kubectl set serviceaccount deployment admin-deployment wap-admin
```
#### Deploy and expose api container
```
kubectl run api-deployment --image=$dockerrepo/whackapod-api --replicas=12 --port=8080 --labels=app=api 
kubectl expose deployment api-deployment --name=api --target-port=8080  --type=NodePort --labels="app=api"
```
#### Deploy and expose game container
```
kubectl run game-deployment --image=$dockerrepo/whackapod-game --replicas=4 --port=8080 --labels=app=game 
kubectl expose deployment game-deployment --name=game --target-port=8080  --type=NodePort --labels="app=game"
```

Verify that your pods are operational by running `kubectl get pods -o wide`. It should look something like below
```
NAME                                READY   STATUS    RESTARTS   AGE     IP            NODE                                NOMINATED NODE   READINESS GATES
admin-deployment-54dbd864bd-9t2ml   1/1     Running   0          7m15s   10.244.1.3    aks-nodepool1-22717319-vmss000000   <none>           <none>
api-deployment-84c56c8bf6-4k2gc     1/1     Running   0          4m2s    10.244.1.9    aks-nodepool1-22717319-vmss000000   <none>           <none>
api-deployment-84c56c8bf6-d7g2t     1/1     Running   0          4m2s    10.244.1.8    aks-nodepool1-22717319-vmss000000   <none>           <none>
api-deployment-84c56c8bf6-gpwxf     1/1     Running   0          4m2s    10.244.1.5    aks-nodepool1-22717319-vmss000000   <none>           <none>P
api-deployment-84c56c8bf6-jctg2     1/1     Running   0          4m2s    10.244.0.11   aks-nodepool1-22717319-vmss000001   <none>           <none>
api-deployment-84c56c8bf6-lhrhr     1/1     Running   0          4m2s    10.244.0.10   aks-nodepool1-22717319-vmss000001   <none>           <none>
api-deployment-84c56c8bf6-r6qnl     1/1     Running   0          4m2s    10.244.0.9    aks-nodepool1-22717319-vmss000001   <none>           <none>
api-deployment-84c56c8bf6-rhq8b     1/1     Running   0          4m2s    10.244.1.4    aks-nodepool1-22717319-vmss000000   <none>           <none>
api-deployment-84c56c8bf6-wch95     1/1     Running   0          4m2s    10.244.0.12   aks-nodepool1-22717319-vmss000001   <none>           <none>
api-deployment-84c56c8bf6-wh5d5     1/1     Running   0          4m2s    10.244.1.10   aks-nodepool1-22717319-vmss000000   <none>           <none>
api-deployment-84c56c8bf6-wzm7x     1/1     Running   0          4m2s    10.244.0.13   aks-nodepool1-22717319-vmss000001   <none>           <none>
api-deployment-84c56c8bf6-xxbjj     1/1     Running   0          4m2s    10.244.1.6    aks-nodepool1-22717319-vmss000000   <none>           <none>
api-deployment-84c56c8bf6-zd9dh     1/1     Running   0          4m2s    10.244.1.7    aks-nodepool1-22717319-vmss000000   <none>           <none>
game-deployment-5d598d6b74-hx7bm    1/1     Running   0          3m9s    10.244.1.12   aks-nodepool1-22717319-vmss000000   <none>           <none>
game-deployment-5d598d6b74-j5qfc    1/1     Running   0          3m9s    10.244.1.11   aks-nodepool1-22717319-vmss000000   <none>           <none>
game-deployment-5d598d6b74-qvvp4    1/1     Running   0          3m9s    10.244.0.14   aks-nodepool1-22717319-vmss000001   <none>           <none>
game-deployment-5d598d6b74-vhbmj    1/1     Running   0          3m9s    10.244.0.15   aks-nodepool1-22717319-vmss000001   <none>           <none>
```
## 3. Publish the game to the internet with an ingress controller
Your application is currently only accessible from inside your Kubernetes cluster. To expose it properly to the internet, we need to deploy an ingress controller. In this example we will deploy and use the [nginx ingress controller](https://docs.microsoft.com/en-us/azure/aks/ingress-basic).

#### Deploy the nginx-ingress with helm
```
#Create namespace
kubectl create namespace ingress-basic

# Add the official stable repository
helm repo add stable https://kubernetes-charts.storage.googleapis.com/

# Use Helm to deploy an NGINX ingress controller
helm install nginx-ingress stable/nginx-ingress \
    --namespace ingress-basic \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
```

#### Create the ingress route (without SSL/TLS)
```
kubectl apply -f https://raw.githubusercontent.com/daltondhcp/whack_a_pod/master/apps/ingress/ingress.aks.yaml
```

Find the external IP address that you will use to access the game with `kubectl get services -n ingress-basic`

```
johan@Azure:~$ kubectl get services -n ingress-basic
NAME                            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
nginx-ingress-controller        LoadBalancer   10.0.196.95    52.167.84.179   80:32735/TCP,443:31620/TCP   17m
nginx-ingress-default-backend   ClusterIP      10.0.169.224   <none>          80/TCP                       17m
```

Point a fancy DNS name to the external IP that you got or try to access it straight with the public ip. 

If you want a less busy version of the game, try /next.html or /advanced.html instead.

## Enjoy the game! :video_game: 

***(Don't forget to tear down your resource group when you are done playing...)***
