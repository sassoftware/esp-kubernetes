## Installation Notes for GCP Kubernetes Service (Google GKE)
This document describes the requirements to
install a SAS Event Stream Processing eco-system in a simple Google Kubernetes Service (Amazon EKS) cluster.  It also provides a specific set of steps to create
a simple Google GKE cluster.

For more detailed information about how to create an Google GKE cluster, please refer to the [documentation](https://cloud.google.com/kubernetes-engine/docs).

**IMPORTANT** To use these notes, you _must_ have an extensive knowledge of Google GKE.

### Required Infrastructure
* A Kubernetes service (Google GKE) 
* A **NGINX** Ingress controller
* A private Domain Name System (DNS) for the Google GKE
* An GOOGLE Virtual Private Cloud (VPC)
* An Goggle Container Registry (GCR) to store SAS Event Stream Processing containers
* The Google  command line tools (gcloud)

To proceed, you must have GCP auth  credentials, be able to
log in to Google Cloud Platform through the [GCP Management Console](https://cloud.google.com/docs/?hl=en_US), and know
how to use the [Google command line tools](https://cloud.google.com/sdk#section-3). 

### Installing SAS Event Stream Processing in a Google GKE Cluster
The following set of scripts are inluded to help create and
manage Google GKE clusters with SAS Event Stream Processing.

---
#### bin/gcp-cluster -- Build a New Google GKE Cluster

This script creates a new Google GKE cluster. The cluster is created in
the geographical location that you specify. An default NFS provider 
is created and associated with the cluster in order to provide a Read Write Many
persistent volume (PV) that you can use for testing.

```
    [bin]$ ./gcp-cluster -?
    Usage: ./bin/gcp-cluster

         required: -c <cluster name>
	 	  -p <gcp project>
                   -g <geographical location>
		   -d <domain for private DNS zone>
                   -f <file to write kubeconfig to>
         optional: -n <number of node, default: 5>
                   -s <vm size, default: e2-standard-16>
```
When it completes, the script reports something like this:
```
Completed build of GKE cluster:""
         GKE cluster:   sckolo-cl
              domain:   gcp.unx.sas.com
     loadbalancer ip:   xxx.yyy.zzz.ttt
    KUBE CONFIG file:   /mnt/data/home/sckolo/scottC-k8.conf
```

---
#### bin/aws-tenant  -- Onboard/Offboard Tenant (Create Namespace and Private DNS record)

This script onboards a tenant to install SAS Event Stream Processing. Specifically, it does the following:

- Creates a namespace in Google GKE with the tenant name
- Creates a record of the form <namespace>.<domain> in the private DNS

```
  [bin]$ ./gcp-tennant -?
  Usage: ./gcp-tennant

      required: -p <GCP project>  -t <tennant name>
      		-c <cluster name>
      optional: -d [delete tennant]
```
When it completes, the script reports something like this:
```
Created kubernetes namespace, private DNS record


You must add an alias record to DNS that points

   <namespace>.<your domain>  --> <GKE loadbalancer IP>

cluster namespace: <namespace>
```

---
#### bin/gcp-push -- Add Docker Images to Google GCR (Creates a Script Named "gcp-images")

This script looks for the following environment variables:
- IMAGE_ESPOAUTH2P
- IMAGE_ESPESM
- IMAGE_ESPSTRMVWR
- IMAGE_OPERATOR
- IMAGE_LOADBAL
- IMAGE_ESPSTUDIO
- IMAGE_METERBILL
- IMAGE_ESPSRV

Each environment variable needs to point to an accessible Docker image. The images are pulled, retagged, and pushed to the Amazon ECR. If the image contains **snapshot** or **release**, then **snapshot/** or **release/** is added to the repository name in the Google GCR.

```
   [bin]$ ./gcp-push  -?
   Usage: ./gcp-push

        required: -p <Google project>

```

---
#### bin/gcp-get-images -- Print Latest images:tags for Repository

This script prints the most recent set of SAS Event Stream Processing images in an Google GKE container registry. The output is in a format that can be cut and pasted into a terminal window in order to set the IMAGE_XXX environment variables. 

```
    [bin]$ ./aws-get-images  -?
    Usage: ./aws-get-images

         required: -p <Google Project>
         optional: -P <prefix for repository> (snapshot | release)
```

---
#### (FOR SAS ONLY) bin/get-images -- Populate IMAGE_XXX Environment Variables from release/snapshot repulpmaster Repository

When sourced (that is, run as: . ./bin/get-images), this script goes to a **SAS repulpmaster** repository and populates the IMAGE_XXX environment with the latest Docker images. 
 
```
    [bin]$ . ./get-images -?
    must be sourced, i.e. run as:

        . ./bin/get-image [-R] | [-S]

    if run as a script: ./bin/get-images [-R] | [-S]
        env variables will not be set!
```

---
### Full Creation of Google GKE Cluster, Onboard Tenant, and Install os ESP

```
$ ./bin/aws -cluster -c sckolo-cl -g us-east-2  -f ~/aws-sckolo-cl-k8.conf
  .
  .
  .
Completed build of EKS cluster:
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

**At this point you must enter an alias into a DNS server. You must point \<tenant name\>.\<domain name\> --> ac55499e0028b4b4eae9026a8b8f9c48-781de1576c00671f.elb.us-east-2.amazonaws.com**

**The \<tenant name\> is arbitrary, the \<domain name\> is governed by your DNS server. The \<tenant name\> and \<domain name\> are used later when deploying the SAS Event Stream Processing application to the Google GKE cluster.**

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

Change to the GitHub esp-kubernetes/esp-cloud project directory.

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
