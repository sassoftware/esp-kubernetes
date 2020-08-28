## Installation notes for Azure AKS

This document does not cover how to install Kubernetes in Azure.
It is intended to describe the Azure environment in which SAS Event Stream Processing
has been installed and tested.

### Infrastucture
* A kubernetes service (AKS)
* An Azure Container Resource to store SAS Event Stream Processing containers
* A private DNS for the AKS 

### Application specifics
* Install the Nginex ingress controller in AKS
* For each namespace in which you plan to deploy SAS Event Stream Processing, create a DNS
record of the name <namespace>.<dns root> with the public IP for the
AKS cluster.
* Look up the username and password for your ACR storage and create
the following secret in each namespace:
```shell
    [AZURE]$ kubectl create secret docker-registry acr-secret --namespace <namespace>
                 --docker-server=espcontainers.azurecr.io
                 --docker-username=<username> --docker-password=<password>
```		 

* Follow the Kubernetes deployment instructions provided in this GitHub repository. Use the
**-A** switch when invoking the **mkdeploy** script. 

