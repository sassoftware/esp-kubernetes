## Installation Notes for Azure Kubernetes Service (AKS)

This document does not attempt to cover in deatail all aspects about
building a AKS cluster in Azure, but hightlights what is required to
stand up a SAS Event Stream Processing eco-system in a simple Azure
AKS cluster. Basic familiarity with Azure is assumed throughout this
note.

### Required Infrastructure
* A Kubernetes service (AKS)
* A **nginx** ingress controller
* A private DNS for the AKS 
* A Microsoft Azure Container Resource to store SAS Event Stream Processing containers

The following steps allow SAS ESP to be easily installed in an AKS
cluster. It is assumed that you already have Azure credentials and can
login to Azure through the Microsoft Azure Portal and use the Azure
command line tools.

Set up the AKS cluster and prepare for the installation. For
illustrative pursposes, these instructions assume that your resource
group is named **azure-esp**, the namespace you are installing into is
named **tennant01** and the dns domain is named **esp.com**. 

* Log into the Microsoft Azure portal and create a new Resource Group to contain all the
required services of your AKS cluster.
* Log in to the microsoft command line and issue the following comand to create the basic cluster. 
```
az aks create --resource-group asuze-esp  --name azureCluster  --node-count 5 \
              --enable-addons monitoring --generate-ssh-keys
```
* use the following command to obtain kubernetes credentials for your
  newly created AKS cluster.
```
az aks get-credentials --resource-group azure-esp  --name azureCluster
```
* Create a namesspace for an **nginx** ingress controller, and install
  the ingress controller using helm.
```
helm install nginx-ingress ingress-nginx/ingress-nginx \
             --namespace ingress-basic --set controller.replicaCount=2 \
	     --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
	     --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux \
	     --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux
```
* To allow secure authentication redirection (oauth2_proxy) to function in your
AKS cluster, a private DNS should be created in your AKS cluster. This is
easily done through the Microsoft Azure Portal. The private DNS has the option to
be bound to a virtual network, and this should be done using the virtual network that is
part of the AKS cluster.
* Still using the Microsoft Azure portal, a **DNS A** record should be
created that points ```<namespace>.<domain>``` to your ingress
external IP. The ingress external ip, can be easily optained using the following command. 
```
kubectl -n ingress-basic get service
```
* In order to install SAS Event Stream Processing into the new AKS
cluster, access to the Docker images for SAS Event Stream Processing
must be available in the AKS cluster. One way to do this, is to create
a Azure Container Registry and push the Docker images into the
cotainer registry. This is done in the same way one pushes docker
images to any docker registry. Azure COntainer regsitries do use
credentials that are required to pull images. Those credentials can be
found on the Microsoft Azure Portal under the container registry
service. One must add these credentials to a Kubernetes secret, before
deploying SAS Event Stream Processing. Enter the credentials into the
*acr-secret** Kubernetes opject as follows:
```
    [AZURE]$ kubectl create secret docker-registry acr-secret --namespace tennant01
                 --docker-server=espcontainers.azurecr.io
                 --docker-username=<username> --docker-password=<password>
```		 
* Follow the Kubernetes deployment instructions provided in this GitHub repository. Use the
**-A** switch when invoking the **mkdeploy** script.


