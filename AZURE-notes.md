## Installation Notes for Azure Kubernetes Service (AKS)

This document highlights the requirements to
install a SAS Event Stream Processing eco-system in a simple Azure Kubernetes Service (AKS) cluster.  It provides a specific set of steps to create
a simple AKS cluster.
For more detailed information about creating an AKS cluster, please refer to the [AKS documentation](https://docs.microsoft.com/en-us/azure/aks/).

To use these notes, you _must_ have a working knowledge of Azure.

### Required Infrastructure
* A Kubernetes service (AKS) 
* A **NGINX** ingress controller
* A private DNS for the AKS 
* A Microsoft Azure Container Resource to store SAS Event Stream Processing containers

### Installing SAS Event Stream Processing in an AKS Cluster
To proceed, you must have Azure credentials, be able to
log in to Azure through the Microsoft Azure Portal, and know how to use the Azure
command line tools.

Suppose that your resource
group is named **azure-esp**, the namespace into which you are installing is
named **tennant01** and the DNS domain is named **esp.com**. 

* Log into the Microsoft Azure portal and create a new Resource Group to contain all the
required services of your AKS cluster.
* Log in to the Microsoft command line and issue the following comand to create the basic cluster. 
```
az aks create --resource-group asuze-esp  --name azureCluster  --node-count 5 \
              --enable-addons monitoring --generate-ssh-keys
```
* Use the following command to obtain Kubernetes credentials for your
  newly created AKS cluster. This action adds access to the cluster through the 
  Kubernetes CLI command line configuration. 
```
az aks get-credentials --resource-group azure-esp  --name azureCluster
```
* Create a namespace for an **NGINX** Ingress controller, and install
  the Ingress controller using Helm.
```
kubectl create ns ingress-basic

helm install nginx-ingress ingress-nginx/ingress-nginx \
             --namespace ingress-basic --set controller.replicaCount=2 \
	     --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
	     --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux \
	     --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux
```
* To enable secure authentication redirection (oauth2_proxy) to function in your
AKS cluster, you need to create a private DNS. Do this through the Microsoft Azure Portal. The private DNS canb
be bound to a virtual network. Perform this binding using the virtual network that is
part of the AKS cluster.
* In the Microsoft Azure portal, create a **DNS A** record that points ```<namespace>.<domain>``` to your Ingress
external IP. You can obtain the Ingress external IP using the following command. 
```
kubectl -n ingress-basic get service
```
* In order to install SAS Event Stream Processing into the new AKS
cluster, make access to the Docker images for SAS Event Stream Processing
available in the AKS cluster. One way to do this is to create
an Azure Container Registry and push the Docker images into the
cotainer registry. Do this the same way that you push Docker
images to any other Docker registry. Azure Container regsitries use credentials that are required to pull images. You can find these credentials on the Microsoft Azure Portal under the container registry
service. You must add these credentials to a Kubernetes secret before
you deploy SAS Event Stream Processing. Enter the credentials into the
*acr-secret** Kubernetes object as follows:
```
    [AZURE]$ kubectl create secret docker-registry acr-secret --namespace tennant01
                 --docker-server=espcontainers.azurecr.io
                 --docker-username=<username> --docker-password=<password>
```		 
* Follow the Kubernetes deployment instructions provided in this GitHub repository. Use the
**-A** switch when invoking the **mkdeploy** script.


