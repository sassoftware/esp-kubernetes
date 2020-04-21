## Event Stream Processing Cloud Ecosystem

### Components
[Operator](/esp-cloud/operator) - Contains YAML template files, and projects to deploy the SAS Event Stream Processing metering server and the ESP operator. 

The following Docker images are deployed from this location:
  * SAS Event Stream Processing metering server
  * ESP operator
  * Open source postgres database
  * Open source filebrowser to manage the persistent volume


[Clients](/esp-cloud/clients) - Contains YAML template files, and projects to deploy SAS Event Stream Processing 
graphics clients.  

The following Docker images are deployed from this location: 
  * SAS Event Stream Processing Studio
  * SAS Event Stream Processing Streamviewer
  * SAS Event Stream Manager

[Oauth2](/esp-cloud/oauth2) - Contains YAML template files for supporting multi-user installs.

The following Docker images are deployed from this location: 
  * SAS Oauth2_proxy
  * Pivitol UAA server

Each of these subdirectories contain README files with more specific, detailed instructions.

### Generating a Deployment

   ./bin/mkdeploy
   Usage: ./bin/mkdeploy

     GENERAL options

          -r                          -- remove existing deploy/
                                          before creating
          -y                          -- no prompt, just execute
          -n <namespace>              -- specify K8 namespace
          -d <ingress domain root>    -- project domain root,
                                          ns.<domain root>/<path> is ingress
          -l <esp license file>       -- SAS ESP license

          -C                          -- deploy clients
          -M                          -- enable multiuser mode

     options for operator deployment

          -o <esp operator image>     -- esp operator docker image
          -s <esp server image>       -- esp server docker image
          -m <esp meter image>        -- esp metering docker image
          -b <sas esp load balancer>  -- esp load balancer docker image

     options for client deployment

          -e <esm image>              -- esp esm docker image
          -t <esp studio image>       -- esp studio docker image
          -v <esp streamviewer image> -- esp streamviewer docker image

     options for multi user user deployment

          -u <uaa server image>       -- opensource UAA server image
          -a <ESP oath2 proxy image>  -- esp authentication proxy image

It is highly reccomended to use the environment variables to specify the docker images, and not pass the docker images on the command line. The enviroment variables one should user are:

```shell
IMAGE_ESPSRV      = "image for SAS Event Stream Processing Server"
IMAGE_LOADBAL     = "image for SAS Event Stream Processing Load Balancer"
IMAGE_METERBILL   = "image for SAS Event Stream Processing Metering Server"
IMAGE_OPERATOR    = "image for SAS Event Stream Processing Operator"

IMAGE_ESPESM      = "image for SAS Event Stream Manager"
IMAGE_ESPSTRMVWR  = "image for SAS Event Stream Processing Streamviewer"
IMAGE_ESPSTUDIO   = "image for SAS Event Stream Processing Studio"

IMAGE_ESPOAUTH2P  = "image for SAS Oauth2 Proxy"
IMAGE_UAA         = "image for Pivitol UAA Server"
```

For example, a full ESP cloud deployment of operator and clients, with mutliuser support would look like:

```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt -n sckolo -d sas.com -C -M
```

**Note:** The *-d* (Ingress domain root) parameter is used to create Ingress routes for the deployed pods.
All Event stream processing components within the kubernetes cluster are now accessed via specific context roots and a single ingress host. The ingress host is of the form `<namespace>.<ingress domain root>`. The following url and context roots are valid:

```
Project X    --   <namespace>.sas.com/SASEventStreamProcessingServer/project/X 
Metering     --   <namespace>.sas.com/SASEventStreamProcessingMetering/SASESP/meterData
Studio       --   <namespace>.sas.com/SASEventStreamProcessingStudio
Streamviewer --   <namespace>.sas.com/SASEventStreamProcessingStreamviewer
ESM          --   <namespace>.sas.com/SASEventStreamManager
FileBrowser  --   <namespace>.sas.com/files
```

For example, if the ingress domain root is `sas.com`, and the namespace is `esp`, then a simple query of metering data would be:
```
     curl http://esp.sas.com:80/SASEventStreamProcessingMetering/eventStreamProcessing/SASESP/meterData
```
for an open deployment, or if protected by a UAA server (multu user deployment)
```
     curl http://esp.sas.com:80/SASEventStreamProcessingMetering/eventStreamProcessing/SASESP/meterData \
     -H 'Authorization: Bearer <put a valid access token here>'
```

### Prerequisite for Deployment, the Persistent Volume 

To deploy the ESP cloud ecosystem, you must have a running Kubernetes cluster and a persistent volume. The persistent volume is used to store the following:

1. the Postgres database
2. Any files (csv/xml/json) referenced by the model running on the ESP server

The Postgres database needs write access to the persistent volume. If one plans on putting other files on the persistent volume, such as CSV input files, or one plans on ESP projects writing files to the persitent volume (output files), the persistent volume must have access mode **ReadWriteMany**. 

If Postgres is the only element of the deployment writing to the persistent volume, it may have access mode **ReadWriteOnce**. 

A typical deployment, with no projects or metadata stored uses about `68MB` of storage. For a typical small deployment, *20GB* of storage for the persistant volume should be adaquate. 
 
After you run ./bin/mkdeploy script and usable deployment manifests are
generated, examine the YAML template file named deploy/pvc.yaml.

```yaml
  #
  # This is the esp-pv claim that esp component pods use.
  #
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
     name: esp-pv
     namespace: esp
  spec:
     accessModes:
       - ReadWriteMany # esp, studio and streamviewer can all write to this space
     resources:
       requests:
         storage: 5Gi  # volume size requested
```

This file specifies the *PersistentVolumeClaim* that the Postgres database, the open source filebrowser app,  and the ESP
server pods make in Kubernetes. 

**Important**: The system administrator must have already set up a persistent volume that can bind to this claim.

In general, the processes associated with the ESP server run user:**sas**, group:**sas**. Commonly, 
this is associated with uid:**1001**, gid:**1001**. An example of this is in the deployment of the open source filebrowser
application.

In the YAML template file deploy/file.yaml. the relevant section is as follows:

```yaml
         initContainers:
         - name: config-data
           image: busybox
           #
           # Our nfs PV is owned by sas:sas which is 1001:1001, so
           #    use those credentials to make <namespace>/{DB.input,output}
           #    directories.
           #
           securityContext:
             runAsUser:  1001
             runAsGroup: 1001
           command: ['sh', '-c', 'mkdir -p /mnt/data/cmdline/input ; mkdir -p /mnt/data/cmdline/output ; touch /db/filebrowser.db']
           volumeMounts:
           - mountPath: /mnt/data
             name: data
```

This specifies an initialization container that runs prior to starting the
filebrowser application. It creates two directories on the persitent volume. 

    input/
    output/

These directories are used by the deployment. The input/ and output/ directories are created
for use by running event stream processing projects that need to access files (csv, xml, etc.).

### Deploying in Kubernetes

After you have revised the manifest files that reside in deploy/, deploy them on the Kubernetes
cluster with the script bin/dodeploy.

```shell
        Usage: ./bin/dodeploy
            -n <namespace>             -- specify K8 namespace
```

Here is a sample invocation of the script bin/dodeploy:

```shell
   ./bin/dodeploy -n cmdline
```

This invocation checks that the given namespace exists before it executes the
deployment. If the namespace does not exist, the script asks whether the namespace should
be created.

After the deployment is completed you should see several active pods in your
namespace:

```
[esp-cloud]$ kubectl -n mudeploy get pods
NAME                                                              READY   STATUS    RESTARTS   AGE
array-685d87f87-mssxr                                             1/1     Running   0          25h
espfb-deployment-5cc85c6bfd-4fb92                                 1/1     Running   0          25h
oauth2-proxy-7d94759449-2thmd                                     1/1     Running   0          25h
postgres-deployment-56c9d65d6c-9k9kb                              1/1     Running   1          25h
sas-esp-operator-5b596f8b6f-wmggk                                 1/1     Running   0          25h
sas-event-stream-manager-app-5b5946b544-9bkxs                     1/1     Running   0          25h
sas-event-stream-processing-client-config-server-75b656f6b7xhpj   1/1     Running   0          25h
sas-event-stream-processing-metering-app-86c5b7c6-c7qv4           1/1     Running   0          24h
sas-event-stream-processing-streamviewer-app-55d79d6996-24vq5     1/1     Running   0          25h
sas-event-stream-processing-studio-app-bf4f675f4-sfpjk            1/1     Running   0          25h
uaa-deployment-85d9fbf6bd-s8fwl                                   1/1     Running   0          25h
```
An Ingress for the each compenent should also appear in the namespace:

```
[esp-cloud]$ kubectl -n mudeploy get ingress
NAME                                               HOSTS            ADDRESS   PORTS     AGE
espfb                                              sckolo.sas.com             80, 443   25h
oauth2-proxy                                       sckolo.sas.com             80, 443   25h
sas-event-stream-manager-app                       sckolo.sas.com             80, 443   25h
sas-event-stream-processing-client-config-server   sckolo.sas.com             80        25h
sas-event-stream-processing-metering-app           sckolo.sas.com             80, 443   24h
sas-event-stream-processing-streamviewer-app       sckolo.sas.com             80, 443   25h
sas-event-stream-processing-studio-app             sckolo.sas.com             80, 443   25h
uaa                                                sckolo.sas.com             80, 443   25h
```

### Using filebrowser

filebrowser is a middleware or standalone app that is available on GitHub.
You can use the filebrowser available with these tools to
access the persistent store used by the Kubernetes pods.  

The filebrowser application is installed in your kubernetes cluster automaticall for convience. It may be accessed
from a browser at:

     http://<namespace>.<ingress domain root>/files

With filebrowser, you can perform the following tasks:

* Copy input files (csv, json, xml) into the persistent store
* View output files written to the persistent store by running projects
* Copy large binary model files for analytics (ASTORE files) to the 
persistent store

