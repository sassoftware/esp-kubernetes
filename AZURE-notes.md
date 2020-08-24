## Installation notes for Azure AKS

We will not attempt to cover the installation of Kubernetes in Azure,
but rather describe the Azure environment that SAS Event Stream Processing
has been install and tested with.

### Infrastucture
* a kubernetes service was created (AKS)
* an Azure Container Ressoure was used to store SAS ESP containers
* a private DNS for the AKS was created

### Application specifics
* install the Nginex ingress controller in AKS
* for each namespace you plan on deploying SAS ESP into, create a DNS
record of the name <namespace>.<dns root> with the public IP for the
AKS cluster.
* Look up the username and password for your ACR storage and create
the following secret in each namespace:
```shell
    [AZURE]$ kubectl create secret docker-registry acr-secret --namespace <namespace>
                 --docker-server=espcontainers.azurecr.io
                 --docker-username=<username> --docker-password=<password>
```		 

* follow noramal ESP kubernetes deployment instructions using the
**-A** switch when invoking the **mkdeploy** script. 

