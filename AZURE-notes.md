## Installation Notes for Azure AKS

This document does explain how to install Kubernetes in Microsoft Azure.
It is intended to describe the Microsoft Azure environment in which you have installed and tested SAS Event Stream Processing.

### Infrastucture
* A kubernetes service (AKS)
* A Microsoft Azure Container Resource to store SAS Event Stream Processing containers
* A private DNS for the AKS 

### Application specifics
* Install the Nginx ingress controller in AKS
* For each namespace in which you plan to deploy SAS Event Stream Processing, create a DNS
record of the name <namespace>.<dns root> with the public IP for the AKS cluster.
* Look up the username and password for your ACR storage. Create
the following secret in each namespace:
```shell
    [AZURE]$ kubectl create secret docker-registry acr-secret --namespace <namespace>
                 --docker-server=espcontainers.azurecr.io
                 --docker-username=<username> --docker-password=<password>
```		 

* Follow the Kubernetes deployment instructions provided in this GitHub repository. Use the
**-A** switch when invoking the **mkdeploy** script. 

