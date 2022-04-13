# Installation Notes for Azure Kubernetes Service (AKS)

This document highlights the requirements to
install a SAS Event Stream Processing eco-system in a simple Azure Kubernetes Service (AKS) cluster.  It provides a specific set of steps to create
a simple AKS cluster.
For more detailed information about creating an AKS cluster, please refer to the [AKS documentation](https://docs.microsoft.com/en-us/azure/aks/).

To use these notes, you _must_ have experience with Azure.

## Required Infrastructure

* A Kubernetes service (AKS)
* A **NGINX** ingress controller
* A private (or public) DNS for the AKS
* A virtual netwrk (AKS will create this for you)
* A Microsoft Azure Container Registry to store SAS Event Stream Processing containers

## Installing SAS Event Stream Processing in an AKS Cluster

To proceed, you must have Azure credentials, be able to
log in to Azure through the Microsoft Azure Portal, and know how to use the Azure
command line tools.

A set of Azure specifics scripts are inluded that can help create and
manage AKS clusters with SAS Event Stream processing.

---

## bin/azure-cluster -- build an AKS cluster from scratch

This command creates a new **AZURE Resource Group** that contains:

* an AKS cluster
* an Azure Container Registry
* a Kubernetes config file for AKS access

The cluster is created in the specified geographical location.

```shell
    [bin]$ ./azure-cluster -?
    Usage: ./bin/azure-cluster

         required: -c <cluster name> -r <resource group>
                   -g <geographical location>
                   -C <container registry>
                   -f <file to write kubeconfig to>
         optional: -n <number of node, default: 5>
                   -s <vm size, default: Standard_F16s_v2>
```

script will report when finished something like:

```text
Completed build resource group: sckoloRG which contains:
         AKS cluster:   sckoloCL
              domain:   b95d519bbe6e4e4c9585.eastus.aksapp.io
   container registy:   sckoloCR
    KUBE CONFIG file:   /mnt/data/home/sckolo/scottC-k8.conf
```

---

## bin/azure-tenant  -- onboard tenant (create ns, and DNS entry)

This script will onboard a tenant for ESP installation. What this translates to is:

* creates a namespace in AKS with the tenant name
* adds a DNS record to AKS to access the tenant eco-system

```shell
  [esp-k8-azure]$ ./bin/azure-tenant -?
  Usage: ./bin/azure-tenant

      required: -C <continer registry> -r <resource group (of cluster)>
                -c <cluster name> -t <tenant name>

      optional: -g <resource group (of container registry>)
```

script will report when finished something like:

```text
cluster host is:   foo.51ebd19c25e24f55b35e.eastus.aksapp.io
cluster namespace: foo
```

---

## bin/azure-startstop  -- start and stop AKS cluster to avoid charges

Start or Stop the AKS cluster. Stopping the cluster with this command avoids being charged by Microsoft. Starting/Stopping the cluster can take several minutes.

```shell
  [bin]$ ./azure-startstop -?
  Usage: ./azure-startstop

       required: -r <resource group> -c <cluster name>
                 -s restart the cluster
                 -d disable cluster (no-cost acrue)
```

---

## bin/azure-push -- add docker images to azure container registry (creates script "asure-images")

This script will look for the following env variables:

* IMAGE_ESPOAUTH2P
* IMAGE_ESPESM
* IMAGE_ESPSTRMVWR
* IMAGE_OPERATOR
* IMAGE_LOADBAL
* IMAGE_ESPSTUDIO
* IMAGE_METERBILL
* IMAGE_ESPSRV

each one should point to an accessable docker image. The images are pulled, retagged, and pushed to the specified Azure Container Registry. If the image contains **snapshot** or **release**, then **snapshot/** or **release/** is added to the repository name in Azure.

```shell
   [bin]$ ./azure-push  -?
   Usage: ./azure-push

        required: -c <continer registry> -r <resource group>

```

---

## bin/azure-purge -- trim container registry to a fixed number of tags

Purge a names container repository to a fixed number of images.

```shell
    [bin]$ ./azure-purge -?
    Usage: ./azure-purge

         required: -c <continer registry> -r <repository name>
                   -k <number of tags to keep>
```

---

## bin/azure-get-images -- print latest images:tags for repository

Print the most recent set of ESP images in an Azure container registry. The output is in a format that can be cut and pasted into a terminal window to set the IMAGE_XXX env variables.

```shell
    [bin]$ ./azure-get-images  -?
    Usage: ./azure-get-images

         required: -c <continer registry> -r <resource group> -R|-S
         optional: -p <prefix for repository>
```

---

## (FOR SAS ONLY) bin/get-images -- populate IMAGE_XXX env vars from release/snapshot repulpmaster repo

This script when sourced (run as: . ./bin/get-images) will go to a **SAS repulpmaster** repository and populate the IMAGE_XXX environment with the latest docker images.

```shell
    [bin]$ . ./get-images -?
    must be sourced, i.e. run as:

        . ./bin/get-image [-R] | [-S]

    if run as a script: ./bin/get-images [-R] | [-S]
        env variables will not be set!
```

## Full creation of Azure cluster, onboard tenant, and install os ESP

```shell
[esp-k8-azure]$ ./bin/azure-cluster -c sckoloCL -r sckoloRG -g eastus -C sckoloCR -f ~/sckoloCL-k8.conf
  .
  .
  .
Completed build resource group: sckoloRG which contains:
         AKS cluster:   sckoloCL
              domain:   41533d7e0a234fdd8d99.eastus.aksapp.io
   container registy:   sckoloCR
    KUBE CONFIG file:   /mnt/data/home/sckolo/sckoloCL-k8.conf
```

```shell
[esp-k8-azure]$ export KUBECONFIG=~/sckoloCL-k8.conf
```

```shell
[esp-k8-azure]$ ./bin/azure-tenant -C sckoloCR -r sckoloRG -c sckoloCL -t esp
  .
  .
  .
cluster host is:   esp.41533d7e0a234fdd8d99.eastus.aksapp.io
cluster namespace: esp
```

```shell
[esp-k8-azure]$ . ./bin/get-images -S
```

```shell
[esp-k8-azure]$ ./bin/azure-push -c sckoloCR -r sckoloRG
  .
  .
  .
 This should push a full set of required docker images into your
 container repository. It will create ./bin/azure images with the
 list of images in exportable format.

Use this command to set al the env variable images in your shell:

[esp-k8-azure]$ . ./bin/azure-images
```

Now change to the github esp-kubernetes/esp-cloud project directory.

```shell
[esp-cloud]$ ./bin/mkdeploy -l ../../LICENSE/setin90.sas -n esp -d 41533d7e0a234fdd8d99.eastus.aksapp.io -r -C -M -A
  .
  .
  .
[esp-cloud]$ ./bin/dodeploy -n esp

[esp-cloud]$ kubectl -n esp get pods
NAME                                                              READY   STATUS    RESTARTS   AGE
espfb-deployment-57c5c4674f-d5x9v                                 1/1     Running   0          2m6s
oauth2-proxy-54d4bff8d9-n9nmq                                     1/1     Running   0          2m1s
postgres-deployment-6fb4b464c6-hlztb                              1/1     Running   0          2m34s
sas-esp-operator-fd4654998-p6cn2                                  1/1     Running   0          2m3s
sas-event-stream-manager-app-6dbb9dcc-fg4wn                       1/1     Running   0          2m2s
sas-event-stream-processing-client-config-server-6b8cfd798hftld   1/1     Running   0          84s
sas-event-stream-processing-metering-app-5df6647b58-ct4wj         1/1     Running   0          2m5s
sas-event-stream-processing-streamviewer-app-5d7478f878-qwq97     1/1     Running   0          2m1s
sas-event-stream-processing-studio-app-755c974645-vp8h7           1/1     Running   0          2m2s
uaa-deployment-94ddb9b48-k9vgp
```

Add the client account:

```shell
    [esp-cloud]$ ./bin/uaatool -u esp.41533d7e0a234fdd8d99.eastus.aksapp.io -C uaaUSER:uaaPASS -c
```

Add a user account:

```shell
    [esp-cloud]$ ./bin/uaatool -u esp.41533d7e0a234fdd8d99.eastus.aksapp.io -C uaaUSER:uaaPASS -a scott:Scott.Kolodzieski@sas.com:scottpw
```
