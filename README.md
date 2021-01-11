## esp-kubernetes

**Note:** The following instructions are specific to SAS Event Stream Processing 2020.1 or later. Older versions of the repository are tagged.

## Changes
For changes between releases, peruse the [changelog](CHANGELOG.md).

## Azure Notes
If you are installing this package on Azure Kubernetes Service (AKS), then read the [Azure Notes](AZURE-notes.md) before deploying SAS Event Stream Processing.

## Introduction
This project is a repository of scripts, YAML template files, and sample projects (XML files) that enable you to develop, deploy, and test an ESP server and SAS Event Stream Processing web-based clients in a Kubernetes cluster.  The resulting SAS Event Stream Processing cloud ecosystem runs independently of SAS Viya.

Use the tools in this repository for either of the following deployment approaches:
* lightweight open, multi-user, multi-tenant deployment
* lightweight open, single-user deployment

Before you proceed:
* Decide which deployment approach you intend to take. Carefully read the associated prerequisites for your chosen deployment before editing any file or running any script.
* Download the pre-built Docker images made available through your SAS Event Stream Processing Software Order Email (SOE).  The SOE points you to information about how to download these images and load them into a local Docker repository.
* Record the names of the images that you download. You specify these names when you set essential environment variables.

## Components of the SAS Event Stream Processing Cloud Ecosystem
### YAML Templates 

The following subdirectories of /esp-cloud provide essential components of the SAS Event Stream Processing cloud ecosystem.

[Operator](/esp-cloud/operator) - contains YAML template files and projects to deploy the ESP operator and the SAS Event Stream Processing metering server. 

From this location, deploy the following Docker images that you obtained through your SOE:
  * SAS Event Stream Processing metering server
  * ESP operator
  * open-source PostgreSQL database (you can replace this with an alternative PostgreSQL database)
  * open-source filebrowser to manage the persistent volume (PV)


[Clients](/esp-cloud/clients) - contains YAML template files and projects to deploy SAS Event Stream Processing 
web-based clients.  

From this location, deploy the following Docker images that you obtained through your SOE: 
  * SAS Event Stream Processing Studio
  * SAS Event Stream Processing Streamviewer
  * SAS Event Stream Manager

[OAuth2](/esp-cloud/oauth2) - contains YAML template files for supporting multi-user installations.

From this location, deploy the following Docker images: 
  * SAS Oauth2_proxy
  * Pivotal User Account and Authentication (UAA) server (configured to store user credentials in PostgreSQL, but could also be reconfigured to read user credentials from alternative identity management (IM) systems)

Each of these subdirectories contains README files with more specific, detailed instructions.

### Scripts

The /bin subdirectory of /esp-cloud provides the following scripts to facilitate deployment:
  * **mkdeploy** - creates a set of deployment YAML files. You must set appropriate environment variables before running this script.
  * **dodeploy** - deploys images on the Kubernetes cluster
  * **mkproject** - converts XML project code into a Kubernetes custom resource file that works in the SAS Event Stream Processing Cloud Ecosystem
  * **uaatool** - allows easy modification (add/delete/list) users in the UAA database used in a multiuser deployment.

For more information about using these scripts, see "Getting Started."
  
## Prerequisites

### Persistent Volume

**Important**: To deploy the Docker images that you have downloaded, you must have a running Kubernetes cluster and two persistent volumes (PVs) available for use.  Work with your Kubernetes administrator to obtain access to a cluster with the required PVs. By default, the persistent volume claims (PVCs) use the Kubernetes storage class "nfs-client" and are dynamically provisioned.  You can change these settings appropriately for your specific installation.

 * The first PV is a backing store for the PostgreSQL database, which requires Write access to the persistent volume. Because the PostgreSQL pod is the only pod that writes to this PV, assign it the access mode **ReadWriteOnce**. A typical deployment with no stored projects or metadata uses about 68MB of storage. For a smaller deployment, 20GB of storage for the PV should be adequate. 
 
 * The second PV is used as a read/write location for running ESP projects.  Because SAS Event Stream Processing projects read and write on the PV simultaneously, you must give it the access mode **ReadWriteMany**.  The size of this PV depends on the amount of input and output data that you intend to store there. Determine the amount of data to be consumed (input data), estimate the amount of processed data to be written (output data), and specify the size accordingly.

### Additional Prerequisites for a Multi-user Deployment
For a multi-user deployment, here are the additional prerequisites:
* access to a Pivotal UAA server in a container
* access to the **cf-uaac** Pivotal UAA command-line client to configure the UAA server

Both of these containers are supplied in the publically available repository:
```
ghcr.io/skolodzieski/uaac            3.2.0      265MB
ghcr.io/skolodzieski/uaa             74.29.0    1.09GB
```

## Getting Started
### Set the Environment Variables

Set the following environment variables before you use the deployment scripts. Use the names that you recorded for the Docker images that you downloaded through the SOE.
```shell
IMAGE_ESPSRV      = "name of image for SAS Event Stream Processing Server"
IMAGE_LOADBAL     = "name of image for SAS Event Stream Processing Load Balancer"
IMAGE_METERBILL   = "name of image for SAS Event Stream Processing Metering Server"
IMAGE_OPERATOR    = "name of image for SAS Event Stream Processing Operator"

IMAGE_ESPESM      = "name of image for SAS Event Stream Manager"
IMAGE_ESPSTRMVWR  = "name of image for SAS Event Stream Processing Streamviewer"
IMAGE_ESPSTUDIO   = "name of image for SAS Event Stream Processing Studio"

IMAGE_ESPOAUTH2P  = "name of image for SAS Oauth2 Proxy"
IMAGE_UAA         = "name of image for Pivotal UAA Server"
```

Perform the SAS Event Stream Processing cloud deployment from a single directory, /esp-cloud. A single script enables the deployment of the ESP operator and the web-based clients. 

The deployment can be performed in Open mode (no TLS or user authentication), or in multi-user mode, which provides full authentication through a UAA server. In multi-user mode, TLS is enabled by default. 

For more information, see [/esp-cloud](/esp-cloud). 

### Location of the Public Domain Images

The deployment makes use of the following third party Docker images:
```
ghcr.io/skolodzieski/busybox       latest
ghcr.io/skolodzieski/filebrowser   latest
ghcr.io/skolodzieski/postgres      12.5 
```

The two files: ```esp-cloud/operator/templates/fileb.yaml```  and ```esp-cloud/operator/templates/postgres.yaml``` reference these docker images and do not need to be modified as long as do not want to replace these third party images.

### Define PostgreSQL/UAA Secrets

The following four enviroment variables control the secrets (created at deployment time) for the Postgres Database and the Pivitol UAA server. 

```
       uaaUsername             --   Username for the UAA server, defaults to uaaUSER (only used in multiuser deployment)
       uaaCredentials          --   Password for the UAA server, defaults to uaaPASS (only used in multiuser deployment)
       postgresSQLUsername     --   Username for the Postgres Database
       postgresSQLCredentials  --   Password for the Postgres Database
```

### Generate a Deployment with mkdeploy

Use the **mkdeploy** script to create a set of deployment YAML files. The script uses the environment variables that you set to locate the Docker images and to pass parameters for specifying a namespace, Ingress root, license, and type of deployment.

   ./bin/mkdeploy
   Usage: ./bin/mkdeploy

     GENERAL options

          -r                          -- remove existing deploy/
                                          before creating
          -y                          -- no prompt, just execute
          -n <namespace>              -- specify K8 namespace
          -d <ingress domain root>    -- project domain root,
                                          ns.<domain root>/<path> is Ingress
          -l <esp license file>       -- SAS ESP license

          -C                          -- deploy clients
          -M                          -- enable multiuser mode

**Note:** Use the *-d* (Ingress domain root) parameter specified in the **mkdeploy** script to create Ingress routes for the deployed pods. All SAS Event Stream Processing applications within the Kubernetes cluster are now accessed through specific context roots and a single Ingress host. The Ingress host is specified in the form `<namespace>.<ingress domain root>`. 
    
The options `-C` and `-M` are optional and generate the following deployments:

* **Open deployment (no authentication or TLS)** with no web-based clients. 
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt \
                            -n <namespace> -d sas.com
```
* **Open deployment (no authentication or TLS)** with all web-based clients. 
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt \
                            -n <namespace> -d sas.com -C
```
* **Multi-user deployment (UAA authentication and TLS)** with no web-based clients.
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt \
                            -n <namespace> -d sas.com -M
```
* **Multi-user deployment (UAA authentication and TLS)** with all web-based clients.
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt \
                            -n <namespace> -d sas.com -C -M
```
After you run the **mkdeploy** script, which generates usable deployment manifests, the YAML template file named deploy/pvc-pg.yaml specifies a *PersistentVolumeClaim*. For example:

```yaml
  #
  # This is the esp-pv claim that esp component pods use.
  #
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
     name: esp-pv-pg
  annotations:
     volume.beta.kubernetes.io/storage-class: "nfs-client"
  spec:
     accessModes:
       - ReadWriteOnce # This volume is used for PostgreSQL storage
     resources:
       requests:
         storage: 20Gi  # volume size requested
```
This *PersistentVolumeClaim* is made by the PostgreSQL database. Ensure that the PV that you have set up can satisfy this claim. 

A second YAML template file named deploy/pvc.yaml specifies a *PersistentVolumeClaim* as the read/write location for running ESP projects. For example:

```yaml
  #
  # This is the esp-pv claim that esp component pods use.
  #
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
     name: esp-pv
  annotations:
     volume.beta.kubernetes.io/storage-class: "nfs-client"
  spec:
     accessModes:
       - ReadWriteMany # This volume is used for ESP projects input/output files
     resources:
       requests:
         storage: 20Gi  # volume size requested
```

In general, the processes associated with the ESP server run user:**sas**, group:**sas**. Typically, 
this choice of user and group means uid:**1001**, gid:**1001**. For example, when you deploy the open source filebrowser
application, the associated processes have these assignments.

In the YAML template file deploy/fileb.yaml, the relevant section is as follows:

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
           command: ['sh', '-c', 'mkdir -p /mnt/data/input ; mkdir -p /mnt/data/output ; touch /db/filebrowser.db']
           volumeMounts:
           - mountPath: /mnt/data
             name: data
```

This section specifies an initialization container that runs prior to starting the
filebrowser application. It creates two directories on the PV: 

    input/
    output/

These directories are used by the deployment. The input/ and output/ directories are created
for use by running event stream processing projects that need to access files (CSV, XML, and so on).


### Deploy Images in Kubernetes with dodeploy

After you have revised the manifest files that reside in the /deploy directory, deploy images on the Kubernetes
cluster with the **dodeploy** script.

```shell
        Usage: ./bin/dodeploy
            -n <namespace>             -- specify K8 namespace
```

For example:

```shell
   ./bin/dodeploy -n cmdline
```

This invocation checks that the given namespace exists before it executes the
deployment. If the namespace does not exist, the script asks whether the namespace should
be created.

After the deployment is completed, you should see active pods within your
namespace. For example, consider the output below. The pods (Ingress) marked with an **M** appear only in a multi-user deployment. The pods (Ingress) marked with a **C** appear only when web-based clients are included in the deployment. 

```
   [esp-cloud]$ kubectl -n mudeploy get pods
   NAME                                                              READY   STATUS    RESTARTS   AGE
   espfb-deployment-5cc85c6bfd-4fb92                                 1/1     Running   0          25h
M  oauth2-proxy-7d94759449-2thmd                                     1/1     Running   0          25h
   postgres-deployment-56c9d65d6c-9k9kb                              1/1     Running   1          25h
   sas-esp-operator-5b596f8b6f-wmggk                                 1/1     Running   0          25h
C  sas-event-stream-manager-app-5b5946b544-9bkxs                     1/1     Running   0          25h
C  sas-event-stream-processing-client-config-server-75b656f6b7xhpj   1/1     Running   0          25h
C  sas-event-stream-processing-metering-app-86c5b7c6-c7qv4           1/1     Running   0          24h
C  sas-event-stream-processing-streamviewer-app-55d79d6996-24vq5     1/1     Running   0          25h
C  sas-event-stream-processing-studio-app-bf4f675f4-sfpjk            1/1     Running   0          25h
M  uaa-deployment-85d9fbf6bd-s8fwl                                   1/1     Running   0          25h
```

The ESP operator, SAS Event Stream Processing Studio, SAS Event Stream Processing Streamviewer, PostgreSQL, oauth2_proxy, and Pivotal UAA are started by the YAML files that are supplied. After SAS Event Stream Processing Studio initializes, it creates a custom resource that causes the ESP operator to start a “client-config-server”, which is a small ESP server that runs a dummy project. SAS Event Stream Processing Studio uses that ESP server to obtain a list of available connectors, algorithms, and other metadata that it requires. 

An Ingress for each component should also appear in the namespace. For example:

```
   [esp-cloud]$ kubectl -n mudeploy get ingress
   NAME                                               HOSTS            ADDRESS   PORTS     AGE
   espfb                                              xxxxxx.sas.com             80, 443   25h
M  oauth2-proxy                                       xxxxxx.sas.com             80, 443   25h
   sas-event-stream-manager-app                       xxxxxx.sas.com             80, 443   25h
C  sas-event-stream-processing-client-config-server   xxxxxx.sas.com             80        25h
C  sas-event-stream-processing--app                   xxxxxx.sas.com             80, 443   24h
C  sas-event-stream-processing-streamviewer-app       xxxxxx.sas.com             80, 443   25h
C  sas-event-stream-processing-studio-app             xxxxxx.sas.com             80, 443   25h
M  uaa                                                xxxxxx.sas.com             80, 443   25h
```

**Note:** The client-config-server shows up last. The remaining pods show up as “running” almost instantaneously.  

## Accessing Projects and Servers 

You can use the following URL and context root to access projects and servers:

```
Project X    --   https://<namespace>.sas.com/SASEventStreamProcessingServer/X/eventStreamProcessing/v1/
Metering     --   https://<namespace>.sas.com/SASEventStreamProcessingMetering/eventStreamProcessing/v1/meterData
Studio       --   https://<namespace>.sas.com/SASEventStreamProcessingStudio
Streamviewer --   https://<namespace>.sas.com/SASEventStreamProcessingStreamviewer
ESM          --   https://<namespace>.sas.com/SASEventStreamManager
filebrowser  --   https://<namespace>.sas.com/files
```

#### Query a Project
Suppose that the Ingress domain root is `sas.com`, the namespace is `esp`, and the project's service name is **array**.  

After deployment, you can query a project deployed in an open environment as follows:
```
     curl https://esp.sas.com/SASEventStreamProcessingServer/array/eventStreamProcessing/v1/
```
You can query a project deployed in a multi-user environment as follows:
```
     curl https://esp.sas.com/SASEventStreamProcessingServer/array/eventStreamProcessing/v1/ \
     -H 'Authorization: Bearer <put a valid access token here>'
```

#### Query the Metering Server
**Note:** You cannot access the metering server through a web browser.

Suppose that the Ingress domain root is `sas.com`, and the namespace is `esp`. 

After deployment, you can perform a simple query of the metering server deployed in an open environment on the cluster as follows:
```
     curl https://esp.sas.com/SASEventStreamProcessingMetering/eventStreamProcessing/v1/meterData
```
You can perform a simple query of the metering server deployed in a multi-user environment on the cluster as follows:
```
     curl https://esp.sas.com/SASEventStreamProcessingMetering/eventStreamProcessing/v1/meterData \
     -H 'Authorization: Bearer <put a valid access token here>'
```


#### Access Web-Based Clients
After deployment, you can access web-based clients in an open or multi-user deployment through the following URLs:
```
SAS Event Stream Processing Studio          -- https://esp.sas.com/SASEventStreamProcessingStudio
SAS Event Stream Processing Streamviewer    -- https://esp.sas.com/SASEventStreamProcessingStreamviewer
SAS Event Stream Processing Manager         -- https://esp.sas.com/SASEventStreamManager
```

## Configuring for Multiple Users

For information about adding service and user accounts and adding credentials, see [Oauth2](/esp-cloud/oauth2).

## Using filebrowser

The filebrowser application on GitHub enables you to access persistent stores used by the Kubernetes pods.  

The filebrowser application is installed in your Kubernetes cluster automatically. You can access it through a browser at the following URL:

     https://<namespace>.<ingress domain root>/files

With filebrowser, you can perform the following tasks:

* copy input files (CSV, JSON, XML) into the persistent store
* view output files written to the persistent store by running projects
* copy large binary model files for analytics (SAS Analytic Store files) to the 
persistent store

## Contributing

We welcome your contributions! Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details about how to submit contributions to this project.

## License

This project is licensed under the [Apache 2.0 License](LICENSE).

## Additional Resources

The [SAS Event Stream Processing product support page](https://support.sas.com/en/software/event-stream-processing-support.html)
contains:
* current and past product documentation
* instructional videos
* examples
* training courses
* featured blogs
* featured community topics


