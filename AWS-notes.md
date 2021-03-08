## Installation Notes for AWS -- Elastic Kubernetes Service (EKS)

This document highlights the requirements to
install a SAS Event Stream Processing eco-system in a simple AWS Kubernetes Service (EKS) cluster.  It provides a specific set of steps to create
a simple EKS cluster.
For more detailed information about creating an AKS cluster, please refer to the [EKS documentation](https://console.aws.amazon.com/eks/).

To use these notes, you _must_ have a working knowledge of AWS.

### Required Infrastructure
* A Kubernetes service (EKS) 
* A **NGINX** ingress controller
* A public DNS for the EKS
* A VPC (EKS will create this for you)
* An AWS Elastic Container Registry to store SAS Event Stream Processing containers


### Installing SAS Event Stream Processing in an EKS Cluster
To proceed, you must have AWS credentials, be able to
log in to AWS through the AWS Management Center, and know
how to use the AWS command line tools. The EKS command line tools
are also required (eksctl).

A set of AWS specifics scripts are inluded that can help create and
manage EKS clusters with SAS Event Stream processing.

---
### bin/aws-cluster -- build an EKS cluster from scratch

This command will create a new EKS cluster. The cluster is created in
the specified geographical location. An AWS Elastic File System (EFS)
is created and associate with the cluster to provide a Read Write Many
persistent volume that may be used for testing.

```
    [bin]$ ./aws-cluster -?
    Usage: ./bin/aws-cluster

         required: -c <cluster name>
                   -g <geographical location>
                   -f <file to write kubeconfig to>
         optional: -n <number of node, default: 5>
                   -s <vm size, default: m5.4xlarge>
```
script will report when finished something like:
```
Completed build resource group: sckoloRG which contains:
         EKS cluster:   sckolo-cl
              domain:   b95d519bbe6e4e4c9585.eastus.aksapp.io
    KUBE CONFIG file:   /mnt/data/home/sckolo/scottC-k8.conf
```

---
### bin/aws-network  -- configure external access to the EKS network

Modifies the security groups for the VPC associated with the EKS
cluster to allow extern access through the ingress host. A CIDR style
IP can be specified (149.173.0.0/16 for SAS internal access [Cary]).

```
  [bin]$ ./aws-network -?
  Usage: ./aws-network

       required: -m <allowed CIDR> -c <cluster name>
                 -g <geographical location>
```

---
### bin/aws-tennant  -- onboard tennant (create ns, EFS access point)

This script will onboard a tennant for ESP installation. What this translates to is:

- creates a namespace in EKS with the tennant name
- create an access point in EFS for the tennant
- create a PV on the access point to be used when deploying to the namespace.

```
  [bin]$ ./aws-tennant -?
  Usage: .aws-tennant

      required: -c <cluster name> -t <tennant name>
                -g <geographical location>
```
script will report when finished something like:
```
Created kubernetes namespace, EFS access point, and
   RXW persitent volume

You must add an alias record to DNS that points

   sckolo.<your domain>  --> "ab627578fd4c14b59bb6f3b3097e740a-c27d8bc8a19ea39a.elb.us-east-2.amazonaws.com"

cluster namespace: sckolo
```

---
### bin/aws-push -- add docker images to elastic container registry (creates script "aws-images")

This script will look for the following env variables:
- IMAGE_ESPOAUTH2P
- IMAGE_ESPESM
- IMAGE_ESPSTRMVWR
- IMAGE_OPERATOR
- IMAGE_LOADBAL
- IMAGE_ESPSTUDIO
- IMAGE_METERBILL
- IMAGE_ESPSRV

each one should point to an accessable docker image. The images are pulled, retagged, and pushed to the AWS Elastic Container Registry. If the image contains **snapshot** or **release**, than **snapshot/** or **release/** is added to the repository name in ECR.

```
   [bin]$ ./aws-push  -?
   Usage: ./aws-push

        required: -g <geographical location>

```

---
### bin/aws-get-images -- print latest images:tags for repository

Print the most recent set of ESP images in an AWS container registry. The ouput is in a format the can be cur/pasted into a terminal window to set the IMAGE_XXX env variables. 

```
    [bin]$ ./aws-get-images  -?
    Usage: ./aws-get-images

         required: -g <geographical location>
         optional: -p <prefix for repository> (snapshot | release)
```

---
### (FOR SAS ONLY) bin/get-images -- populate IMAGE_XXX env vars from release/snapshot repulpmaster repo

This script when sourced (run as: . ./bin/get-images) will go to a **SAS repulpmaster** reposiory and populate the IMAGE_XXX environment with the latest docker images. 
 
```
    [bin]$ . ./get-images -?
    must be sourced, i.e. run as:

        . ./bin/get-image [-R] | [-S]

    if run as a script: ./bin/get-images [-R] | [-S]
        env variables will not be set!
```

---
## Full creation of AWS EKS cluster, onboard tennant, and install os ESP

```
$ ./bin/aws -cluster -c sckolo-cl -g us-east-2  -f ~/aws-sckolo-cl-k8.conf
  .
  .
  .
Completed build od EKS cluster:
         EKS cluster:   sckolo-cl
              domain:   ac55499e0028b4b4eae9026a8b8f9c48-781de1576c00671f.elb.us-east-2.amazonaws.com
    KUBE CONFIG file:   /mnt/data/home/sckolo/AWS.yaml
```

```
$ export KUBECONFIG=~/aws-sckolo-cl-k8.conf
```

```
$ ./bin/aws-network -c sckolo-cl  -m 149.173.0.0/16 -g us-east-2
nodegroup name is ng-188f74b1
securitygroup tag name is eksctl-sckolo-cl-nodegroup-ng-188f74b1
securitygroup ID is sg-0102d2ea94dea287d  .
  .
  .
  .
```

```
./bin/aws-tennant  -c sckolo-cl -t sckolo -g us-east-2
  .
  .
  .
Created kubernetes namespace, EFS access point, and
   RXW persitent volume

You must add an alias record to DNS that points

   sckolo.<your domain>  --> "ab627578fd4c14b59bb6f3b3097e740a-c27d8bc8a19ea39a.elb.us-east-2.amazonaws.com"

cluster namespace: sckolo
```

**At this point you need to enter an alias into a DNS server. You need to point \<tennant name\>.\<domain name\> --> ac55499e0028b4b4eae9026a8b8f9c48-781de1576c00671f.elb.us-east-2.amazonaws.com**

**The \<tennnant name\> is fairly arbitrary, the \<domain name\> is governed by your DNS server. The \<tennant name\> and \<domain name\> will be used later when deploying the ESP application to the EKS cluster.**

```
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

Now change to the github esp-kubernetes/esp-cloud project directory.

```
$ ./bin/mkdeploy -l ../../LICENSE/setin90.sas -n <tennant name> -d <domain name> -r -C -M -W
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
```
    [esp-cloud]$ ./bin/uaatool -u esp.41533d7e0a234fdd8d99.eastus.aksapp.io -C uaaUSER:uaaPASS -c
```
Add a user account:
```
    [esp-cloud]$ ./bin/uaatool -u esp.41533d7e0a234fdd8d99.eastus.aksapp.io -C uaaUSER:uaaPASS -a scott:Scott.Kolodzieski@sas.com:scottpw
```
