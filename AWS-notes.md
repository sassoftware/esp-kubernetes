# Installation Notes for Amazon Elastic Kubernetes Service (Amazon EKS)

This document describes the requirements to
install a SAS Event Stream Processing eco-system in a simple Amazon Elastic Kubernetes Service (Amazon EKS) cluster.  It also provides a specific set of steps to create
a simple Amazon EKS cluster.

For more detailed information about how to create an Amazon EKS cluster, please refer to the [documentation](https://console.aws.amazon.com/eks/).

**IMPORTANT** To use these notes, you _must_ have experience with Amazon EKS.

## Required Infrastructure

* A Kubernetes service (Amazon EKS)
* A **NGINX** Ingress controller
* A public Domain Name System (DNS) for the Amazon EKS
* An Amazon Virtual Private Cloud (VPC) (Amazon EKS creates this for you)
* An Amazon Elastic Container Registry (ECR) to store SAS Event Stream Processing containers
* The Amazon EKS command line tools (eksctl)

To proceed, you must have Amazon EKS credentials, be able to
log in to Amazon EKS through the [AWS Management Console](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html), and know
how to use the [Amazon EKS command line tools](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html).

## Installing SAS Event Stream Processing in an Amazon EKS Cluster

The following set of scripts are included to help create and
manage Amazon EKS clusters with SAS Event Stream Processing.

---

### bin/aws-cluster -- Build a New Amazon EKS Cluster

This script creates a new Amazon EKS cluster. The cluster is created in
the geographical location that you specify. An Amazon Elastic File System (Amazon EFS)
is created and associated with the cluster in order to provide a Read Write Many
persistent volume (PV) that you can use for testing.

```shell
    [bin]$ ./aws-cluster -?
    Usage: ./bin/aws-cluster

         required: -c <cluster name>
                   -g <geographical location>
                   -f <file to write kubeconfig to>
         optional: -n <number of node, default: 5>
                   -s <vm size, default: m5.4xlarge>
```

When it completes, the script reports something like this:

```text
Completed build resource group: sckoloRG which contains:
         EKS cluster:   sckolo-cl
              domain:   b95d519bbe6e4e4c9585.eastus.aksapp.io
    KUBE CONFIG file:   /mnt/data/home/sckolo/scottC-k8.conf
```

---

### bin/aws-network  -- Configure External Access to the Amazon EKS Network

This script modifies the security groups for the Amazon Virtual Private Cloud (VPC) that is associated with the Amazon EKS
cluster. This enables external access through the Ingress host. You can specify a Classless Inter-Domain Routing (CIDR) style
IP.

<!--(149.173.0.0/16 for SAS internal access [Cary]).-->

```shell
  [bin]$ ./aws-network -?
  Usage: ./aws-network

       required: -m <allowed CIDR> -c <cluster name>
                 -g <geographical location>
```

For example:

```shell
./aws-network -c sckolo-cl  -m x.x.x.x/x -g us-east-2
```

---

### bin/aws-tenant  -- Onboard Tenant (Create Namespace and EFS Access Point)

This script onboards a tenant to install SAS Event Stream Processing. Specifically, it does the following:

* Creates a namespace in Amazon EKS with the tenant name
* Creates an access point in EFS for the tenant
* Creates a PV on the access point to be used when deploying to the namespace

```shell
  [bin]$ ./aws-tenant -?
  Usage: .aws-tenant

      required: -c <cluster name> -t <tenant name>
                -g <geographical location>
```

When it completes, the script reports something like this:

```text
Created Kubernetes namespace, EFS access point, and
   RXW persistent volume

You must add an alias record to DNS that points

   sckolo.<your domain>  --> "ab627578fd4c14b59bb6f3b3097e740a-c27d8bc8a19ea39a.elb.us-east-2.amazonaws.com"

cluster namespace: sckolo
```

---

### bin/aws-push -- Add Docker Images to Amazon ECR (Creates a Script Named "aws-images")

This script looks for the following environment variables:

* IMAGE_ESPOAUTH2P
* IMAGE_ESPESM
* IMAGE_ESPSTRMVWR
* IMAGE_OPERATOR
* IMAGE_ESPSTUDIO
* IMAGE_METERBILL
* IMAGE_ESPSRV

Each environment variable needs to point to an accessible Docker image. The images are pulled, retagged, and pushed to the Amazon ECR. If the image contains **snapshot** or **release**, then **snapshot/** or **release/** is added to the repository name in the Amazon ECR.

```shell
   [bin]$ ./aws-push  -?
   Usage: ./aws-push

        required: -g <geographical location>

```

---

### bin/aws-get-images -- Print Latest images:tags for Repository

This script prints the most recent set of SAS Event Stream Processing images in an Amazon EKS container registry. The output is in a format that can be cut and pasted into a terminal window in order to set the IMAGE_XXX environment variables.

```shell
    [bin]$ ./aws-get-images  -?
    Usage: ./aws-get-images

         required: -g <geographical location>
         optional: -p <prefix for repository> (snapshot | release)
```

---

### (FOR SAS ONLY) bin/get-images -- Populate IMAGE_XXX Environment Variables from release/snapshot repulpmaster Repository

When sourced (that is, run as: . ./bin/get-images), this script goes to a **SAS repulpmaster** repository and populates the IMAGE_XXX environment with the latest Docker images.

```shell
    [bin]$ . ./get-images -?
    must be sourced, i.e. run as:

        . ./bin/get-image [-R] | [-S]

    if run as a script: ./bin/get-images [-R] | [-S]
        env variables will not be set!
```

---

## Full Creation of Amazon EKS Cluster, Onboard Tenant, and Install os ESP

```shell
$ ./bin/aws -cluster -c sckolo-cl -g us-east-2  -f ~/aws-sckolo-cl-k8.conf
  .
  .
  .
Completed build of EKS cluster:
         EKS cluster:   sckolo-cl
              domain:   ac55499e0028b4b4eae9026a8b8f9c48-781de1576c00671f.elb.us-east-2.amazonaws.com
    KUBE CONFIG file:   /mnt/data/home/sckolo/AWS.yaml
```

```shell
export KUBECONFIG=~/aws-sckolo-cl-k8.conf
```

```shell
$ ./bin/aws-network -c sckolo-cl  -m 149.173.0.0/16 -g us-east-2
nodegroup name is ng-188f74b1
securitygroup tag name is eksctl-sckolo-cl-nodegroup-ng-188f74b1
securitygroup ID is sg-0102d2ea94dea287d  .
  .
  .
  .
```

```shell
./bin/aws-tenant  -c sckolo-cl -t sckolo -g us-east-2
  .
  .
  .
Created Kubernetes namespace, EFS access point, and
   RXW persitent volume

You must add an alias record to DNS that points

   sckolo.<your domain>  --> "ab627578fd4c14b59bb6f3b3097e740a-c27d8bc8a19ea39a.elb.us-east-2.amazonaws.com"

cluster namespace: sckolo
```

**At this point you must enter an alias into a DNS server.
You must point \<tenant name\>.\<domain name\> --> ac55499e0028b4b4eae9026a8b8f9c48-781de1576c00671f.elb.us-east-2.amazonaws.com**

**The \<tenant name\> is arbitrary, the \<domain name\> is governed by your DNS server. The \<tenant name\> and \<domain name\> are used later when deploying the SAS Event Stream Processing application to the Amazon EKS cluster.**

```shell
$ . ./bin/get-images -S
$ ./bin/aws-push -g us-east-2
  .
  .
  .
 This should push a full set of required docker images into your
 container repository. It will create ./bin/azure images with the
 list of images in exportable format.

Use this command to set al the env variable images in your shell:

$ ./bin/aws-get-images -g us-east-2 -p snapshot
$ . ./bin/aws-images
```

Change to the GitHub esp-kubernetes/esp-cloud project directory.

```shell
$ ./bin/mkdeploy -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt -n <tenant name> -d <domain name> -r -C -M -W
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
