## Installation Notes for Azure Kubernetes Service (AKS)

This document highlights the requirements to
install a SAS Event Stream Processing eco-system in a simple Azure Kubernetes Service (AKS) cluster.  It provides a specific set of steps to create
a simple AKS cluster.
For more detailed information about creating an AKS cluster, please refer to the [AKS documentation](https://docs.microsoft.com/en-us/azure/aks/).

To use these notes, you _must_ have a working knowledge of Azure.

### Required Infrastructure
* A Kubernetes service (AKS) 
* A **NGINX** ingress controller
* A private (or public) DNS for the AKS
* A virtual netwrk (AKS will create this for you)
* A Microsoft Azure Container Registry to store SAS Event Stream Processing containers

### Installing SAS Event Stream Processing in an AKS Cluster
To proceed, you must have Azure credentials, be able to
log in to Azure through the Microsoft Azure Portal, and know how to use the Azure
command line tools.

The following example script can be modified and used to create the AKS.

```shell
#!/bin/bash

#
# helm much be on your path!
#
az account show &>/dev/null ; [ 0 -ne $? ] && echo "not logged in to azure" && exit 1
which helm &>/dev/null      ; [ 0 -ne $? ] && echo "helm not found on path" && exit 1


# To build a basic Azure K8 cluster, define the following ENV variables:
#
#  1. AZ_RG -- the azure resource group to create,
#              this holds all Azure objects as a parent container.
#
#  2. AZ_CL -- the K8 cluster name
#
#  3. AZ_LO -- geographical location of the secondary resource group
#              that Azure creates for you.
#
#  4. AZ_DM -- the domain used for the private DNS
#
#  5. AZ_NS -- the namespace that will be used to install, also for the
#              private-dns A record: $AZ_NS.$AZ_DM --> public ip
#

#
# When we write the credentials for the newly created AKS cluster
#
KUBE_CRED_FILE=~/AZ_CUBE.config

AZ_RG=scottRG
AZ_CL=scottCluster
AZ_LO=eastus
AZ_DM=azure.esp.com
AZ_NS=sckolo


AZ_NODE_NUM=5
#
# Standard_F16s_v2	16 cores	32 GB ram
#
# The Fsv2-series runs on the Intel® Xeon® Platinum 8272CL (Cascade
#    Lake) processors and Intel® Xeon® Platinum 8168 (Skylake)
#    processors. It features a sustained all core Turbo clock speed of 3.4
#    GHz and a maximum single-core turbo frequency of 3.7 GHz.
#
# do "az vm list-sizes -o table -l eastus"
#
#    to see VM's that are available.
#
AZ_NODE_SIZE=Standard_F16s_v2


echo "We are going to build an Azure K8 cluster with the following parameters"
echo "  (all will be created, and should not already exist):"
echo
echo "  resource group:   $AZ_RG"
echo "  cluster name:     $AZ_CL"
echo "  cluster location: $AZ_LO"
echo "  cluster domain:   $AZ_DM"
echo "  cluster namespce: $AZ_NS"
echo "     # of nodes:    $AZ_NODE_NUM"
echo "     node type:     $AZ_NODE_SIZE"
echo "  K8 config file:   $KUBE_CRED_FILE"
echo
read -p "Continue (yes/[no])? " ANS
[ "$ANS" = "yes" ] || exit 1



#
# create the top level resource group that contains everything
#
az group create --location $AZ_LO --name $AZ_RG

#
# create a basic 5 node cluster
#
az aks create --resource-group $AZ_RG --name $AZ_CL \
              --node-count $AZ_NODE_NUM --node-vm-size $AZ_NODE_SIZE \
              --enable-addons monitoring --generate-ssh-keys
az aks wait --created  --resource-group $AZ_RG --name $AZ_CL

#
# get the K8 credentials for the new cluster into our kubeconfig
#
az aks get-credentials --resource-group $AZ_RG  --name $AZ_CL --file $KUBE_CRED_FILE --overwrite-existing
export KUBECONFIG=$KUBE_CRED_FILE

#
# create the namespace for nginx
#
kubectl create ns nginx-basic

#
# create the namespace for ESP pods
#
kubectl create ns $AZ_NS

#
# use helm to install nginx
#
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx &> /dev/null
[ 0 -ne $? ] && echo "could not add the ingress-nginx helm repository" && exit 1

helm install nginx-ingress ingress-nginx/ingress-nginx \
             --namespace nginx-basic --set controller.replicaCount=2 \
	     --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
	     --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux \
	     --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux
#
# create the private DNS zone
#
az network private-dns zone create --name $AZ_DM --resource-group $AZ_RG
az network private-dns zone wait --created --name $AZ_DM --resource-group $AZ_RG

#
# get public IP into AZ_PIP
#
AZ_PIP=`kubectl -n nginx-basic get svc/nginx-ingress-ingress-nginx-controller --output jsonpath='{.status.loadBalancer.ingress[0].ip}'`

#
# add the A record into the private DNS
#
az network private-dns record-set a add-record -g $AZ_RG -z $AZ_DM -n $AZ_NS -a $AZ_PIP

#
# get the virtual network
#
AZ_VN=`az network vnet list --resource-group MC_"$AZ_RG"_"$AZ_CL"_"$AZ_LO" | jq .[].name`
AZ_VN=`echo $AZ_VN | xargs`    ; # remove double quotes.
AZ_VNID=`az network vnet show -g MC_"$AZ_RG"_"$AZ_CL"_"$AZ_LO" -n $AZ_VN --query 'id' -o tsv`

#
# link the virtual network of the K8 custer and the private-dns
#
az network private-dns link vnet create -g $AZ_RG -n pDNS2vNet -z $AZ_DM -v $AZ_VNID -e False

echo
echo "********************************************************************"
echo "1. add the following to your /etc/hosts file"
echo
echo "    $AZ_PIP $AZ_NS.$AZ_DM"
echo
echo
echo "2. to use the uaatool to configure the UAA autentication server, set"
echo
echo "    export DOCKER_ARGS=\"--add-host=$AZ_PIP $AZ_NS.$AZ_DM:$AZ_PIP\""
echo "********************************************************************"
```


* In order to install SAS Event Stream Processing into the new AKS
cluster, access to the Docker images for SAS Event Stream Processing
must be available in the AKS cluster. One way to do this is to create
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


