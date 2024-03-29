# Installation Notes for Google Kubernetes Engine (GKE)

This document describes the requirements to
install a SAS Event Stream Processing eco-system in a simple Google Kubernetes Engine (GKE) cluster.  It also provides a specific set of steps to create
a simple GKE cluster.

For more detailed information about how to create an GKE cluster, please refer to the [documentation](https://cloud.google.com/kubernetes-engine/docs).

**IMPORTANT** To use these notes, you _must_ have experience with Google Kubernetes Engine (GKE).

## Required Infrastructure

* A Kubernetes service (GKE)
* A **NGINX** Ingress controller
* A private Domain Name System (DNS) for the Google GKE
* A Google Virtual Private Cloud (VPC)
* An Google Container Registry (GCR) to store SAS Event Stream Processing containers
* The Google command line tools (gcloud)

To proceed, you must have gcp auth method credentials, be able to
log in to Google Cloud Platform (GCP) through the [GCP Management Console](https://cloud.google.com/docs/?hl=en_US), and know
how to use [Google command line tools](https://cloud.google.com/sdk#section-3).

## Installing SAS Event Stream Processing in a Google Kubernetes Engine (GKE) Cluster

The following set of scripts are included to help create and
manage GKE clusters with SAS Event Stream Processing.

---

### bin/gcp-cluster -- Build a New GKE Cluster

This script creates a new GKE cluster. The cluster is created in
the geographical location that you specify. A default NFS provider
is created and associated with the cluster in order to provide a Read Write Many
persistent volume (PV) that you can use for testing.

```shell
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

```text
Completed build of GKE cluster:""
         GKE cluster:   sckolo-cl
              domain:   gcp.sas.com
     loadbalancer ip:   xxx.yyy.zzz.ttt
    KUBE CONFIG file:   /mnt/data/home/sckolo/scottC-k8.conf
```

---

#### bin/aws-tenant  -- Onboard/Offboard Tenant (Create Namespace and Private DNS record)

This script onboards a tenant to install SAS Event Stream Processing. Specifically, it does the following:

* Creates a namespace in GKE with the tenant name
* Creates a record of the form `<namespace>.<domain>` in the private DNS

```shell
  [bin]$ ./gcp-tenant -?
  Usage: ./gcp-tenant

      required: -p <GCP project>  -t <tenant name>
                -c <cluster name>
      optional: -d [delete tenant]
```

When it completes, the script reports something like this:

```text
Created Kubernetes namespace, private DNS record


You must add an alias record to DNS that points

   <namespace>.<your domain>  --> <GKE loadbalancer IP>

cluster namespace: <namespace>
```

---

### bin/gcp-push -- Add Docker Images to Google Container Registry (GCR) (Creates a Script Named "gcp-images")

This script looks for the following environment variables:

* IMAGE_ESPESM
* IMAGE_ESPSTRMVWR
* IMAGE_OPERATOR
* IMAGE_ESPSTUDIO
* IMAGE_METERBILL
* IMAGE_ESPSRV
* IMAGE_OAUTH2P

Each environment variable needs to point to an accessible Docker image. The images are pulled, re-tagged, and pushed to the GCR. If the image contains **snapshot** or **release**, then **snapshot/** or **release/** is added to the repository name in the GCR.

```shell
   [bin]$ ./gcp-push  -?
   Usage: ./gcp-push

        required: -p <Google project>

```

---

#### bin/gcp-get-images -- Print Latest images:tags for Repository

This script prints the most recent set of SAS Event Stream Processing images in a GCR. The output is in a format that can be cut and pasted into a terminal window in order to set the IMAGE_XXX environment variables.

```shell
    [bin]$ ./aws-get-images  -?
    Usage: ./aws-get-images

         required: -p <Google Project>
         optional: -P <prefix for repository> (snapshot | release)
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

## Full Creation of GKE Cluster, Onboard Tenant, and Install of ESP

```shell
$ ./bin/gcp-tenant -c sckolo-cl -p solorgasub1 -t sckolo
creating K8s namespace
namespace/sckolo created
   .
   .
   .
Created Kubernetes namespace, private DNS record

You must add an alias record to DNS that points

   sckolo.gcp.sas.com  --> 35.193.42.103

cluster namespace: sckolo
```

**At this point you must enter an alias into a DNS server.
You must point \<tenant name\>.\<domain name\> to 35.193.42.103**

```shell
$ . ./bin/get-images -S
$ ./bin/gcp-push -p solorgasub1
  .
  .
  .
 This should push a full set of required docker images into your
 container repository. It will create ./bin/gcp-images with the
 list of images in exportable format.

Use this command to set al the env variable images in your shell:

$ ./gcp-get-images -P snapshot -p solorgasub1
$ . ./bin/gcp-images
```

Change to the GitHub esp-kubernetes/esp-cloud project directory.

```shell
$ ./bin/mkdeploy -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt -n <tenant name> -d <domain name> -r -C -M -G
  .
  .
  .
$ ./bin/dodeploy -n esp

$ kubectl -n esp get pods
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

Add a user account:

```shell
./bin/uaatool -u sckolo.gcp.sas.com -C uaaUSER:uaaPASS -a sk:scott.kolodzieski@sas.com:sckoloPW1
```
