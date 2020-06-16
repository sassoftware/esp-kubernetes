## esp-kubernetes

**Note:** These instructions are specific to SAS Event Stream Processing 7.1 or later.

## Changes
For changes between releases, read the [changelog](CHANGELOG.md).

## Introduction
This project is a repository of scripts, YAML template files, and sample projects (XML files) that enable you to develop, deploy, and test an ESP server and SAS Event Stream Processing web-based clients in a Kubernetes cluster.  They apply to the following deployment approaches:
* lightweight open, multi-user, multi-tenant deployment
* lightweight open, single-user deployment

Before you proceed:
* Decide which of these approaches you intend to take. Read the associated prerequisites before getting started.
* Download the pre-built Docker images made available through your SAS Event Stream Processing Software Order Email (SOE).  The SOE guides you to information about how to download these images and load them onto a local Docker repository.

## Components of the SAS Event Stream Processing Cloud Ecosystem

The Docker images provide the following components of the SAS Event Stream Processing Cloud Ecosystem.

[Operator](/esp-cloud/operator) - contains YAML template files, and projects to deploy the SAS Event Stream Processing metering server and the ESP operator. 

The following Docker images are deployed from this location:
  * SAS Event Stream Processing metering server
  * ESP operator
  * open source PostgreSQL database (This could be replaced by an alternative PostgreSQL database.)
  * open source file browser to manage the persistent volume


[Clients](/esp-cloud/clients) - contains YAML template files, and projects to deploy SAS Event Stream Processing 
web-based clients.  

The following Docker images are deployed from this location: 
  * SAS Event Stream Processing Studio
  * SAS Event Stream Processing Streamviewer
  * SAS Event Stream Manager

[OAuth2](/esp-cloud/oauth2) - contains YAML template files for supporting multi-user installations.

The following Docker images are deployed from this location: 
  * SAS Oauth2_proxy
  * Pivotal User Account and Authentication (UAA) server (configured to store user credentials in PostgreSQL, but could be reconfigured to read user credentials from alternative identity management (IM) systems)

Each of these subdirectories contains README files with more specific, detailed instructions.

## Prerequisites

### Persistent Volume

**Important**: To deploy the images, you must have a running Kubernetes cluster and a persistent volume available for use.  Work with your Kubernetes administrator to obtain access to a cluster with a persistent volume.

The persistent volume is used to store the following:

1. The PostgreSQL database
2. Any files (CSV/XML/JSON) referenced by the model that is running on the ESP server

The PostgreSQL database requires Write access to the persistent volume. If you plan to put other files on the persistent volume, such as CSV input files, or plan for projects to write files to the persistent volume (output files), the persistent volume must have the access mode **ReadWriteMany**.  

If PostgreSQL is the only element of the deployment that writes to the persistent volume, it can have the access mode **ReadWriteOnce**. 

A typical deployment, with no stored projects or metadata, uses about 68MB of storage. For a typical small deployment, 20GB of storage for the persistent volume should be adaquate. 
 
After you run the ./bin/mkdeploy script, which generates usable deployment manifests, the YAML template file named deploy/pvc.yaml specifies a *PersistentVolumeClaim*.

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
         storage: 20Gi  # volume size requested
```

This *PersistentVolumeClaim* is made by the PostgreSQL database, the open source filebrowser application, and the ESP
server pods in the Kubernetes environment. Ensure that the persistent volume that you have set up can satisfy this claim. 

In general, the processes associated with the ESP server run user:**sas**, group:**sas**. Commonly, 
this is associated with uid:**1001**, gid:**1001**. An example of this is in the deployment of the open source filebrowser
application.

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
           command: ['sh', '-c', 'mkdir -p /mnt/data/cmdline/input ; mkdir -p /mnt/data/cmdline/output ; touch /db/filebrowser.db']
           volumeMounts:
           - mountPath: /mnt/data
             name: data
```

This section specifies an initialization container that runs prior to starting the
filebrowser application. It creates two directories on the persistent volume: 

    input/
    output/

These directories are used by the deployment. The input/ and output/ directories are created
for use by running event stream processing projects that need to access files (CSV, XML, and so on).

### Additional Prerequisites for a Multi-user Deployment
For a multi-user deployment, here are the following additional prerequisites:
* access to a Pivotal UAA server in a container
* access to the "uaac" Pivotal UAA command-line tool to configure the UAA server.

To create your own UAA Docker container, download a recent UAA WAR file (such as cloudfoundry-identity-uaa-4.30.0.war) from any Maven repository and use the following Docker file:

```
FROM tomcat:8-jre8-alpine

ENV CATALINA_OPTS="-Xmx800m"

RUN rm $CATALINA_HOME/webapps/ROOT -r -f
ADD cloudfoundry-identity-uaa-4.30.0.war $CATALINA_HOME/webapps/uaa.war

EXPOSE 8080
```
A convenient way to run the uaac command line client is to build a Docker container with just the uaac client.
Use the following Docker file:
```
FROM ruby:2.6-alpine3.9

# TODO: remove after https://github.com/docker-library/ruby/pull/209 was fixed.
ENV PATH "/usr/local/bundle/bin:${PATH}"

RUN apk add --no-cache musl-dev gcc make g++

RUN gem install cf-uaac -v 3.2.0 --no-document
```

## Getting Started
### Set Environment Variables

It is recommended that you set the following environment variables before you use the deployment scripts:
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
Alternatively, you can specify the location of the images on the command line.

Perform the SAS Event Stream Processing cloud deployment from a single directory, esp-cloud. A single script enables the deployment of the ESP operator and the graphical clients. 

The deployment can be performed in Open mode (no TLS or user authentication), or in multi-user mode, which provides full authentication through a UAA server. In multi-user mode, TLS is enabled by default. 

For more information, see [ESP cloud](/esp-cloud). 

### Generate a Deployment

Use the mkdeploy script to create a set of deployment YAML files. The script uses the environment variables that you set to locate the Docker images and pass parameters to specify a namespace, Ingress root, license, and type of deployment.

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

**Note:** Use the *-d* (Ingress domain root) parameter specified in the mkdeploy command to create Ingress routes for the deployed pods. All SAS Event Stream Processing applications within the Kubernetes cluster are now accessed through specific context roots and a single Ingress host. The Ingress host is of the form `<namespace>.<ingress domain root>`. 
    
The options `-C` and `-M` are optional and generate the following deployments:

* **Open deployment (no authentication or TLS)** with no graphical clients. 
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt \
                            -n sckolo -d sas.com
```
* **Open deployment (no authentication or TLS)** with all graphical clients. 
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt \
                            -n sckolo -d sas.com -C
```
* **Multi-user deployment (UAA authentication and TLS)** with no graphical clients.
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt \
                            -n sckolo -d sas.com -M
```
* **Multi-user deployment (UAA authentication and TLS)** with all graphical clients.
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt \
                            -n sckolo -d sas.com -C -M
```

### Define Postgres Secrets and Management

The **mkdeploy** script creates the file `esp-cloud/deploy/postgres.yaml`. The first few lines define the username and 
password for the admin account in Postgres. 
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
data:
  username: ZXNw
  password: ZXNwX2luX2Nsb3Vk
```
These are base64 encoded strings. The default values are **esp** for the username and **esp_in_cloud** for the password. They can be adjusted prior to deployment. 

After the deployment is successful and the Postgres pod has started, the Postgres instance can be administered with **psql** from the Kubernetes cluster. Use the following command to connect to psql: 

```shell
kubectl -n sckolo exec -it postgres-deployment-6f9d6cc8cc-mhx79 -- psql -U esp
```

**Note:** The name of your PostreSQL pod will differ from **postgres-deployment-6f9d6cc8cc-mhx79**. 


### Deploy Images in Kubernetes

After you have revised the manifest files that reside in the deploy/ directory, deploy images on the Kubernetes
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

After the deployment is completed, you should see active pods within your
namespace. For example, consider the output below. The pods (Ingress) marked with a **M** appear only in a multi-user deployment. The pods (Ingress) marked with a **C** appear only when graphical clients are included in the deployment. 

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

The ESP operator, SAS Event Stream Processing Studio, SAS Event Stream Processing Streamviewer, PostgreSQL, oauth2_proxy, and Pivotal UAA are started by the YAML files supplies. After SAS Event Stream Processing Studio initializes, it creates a custom resource that causes the ESP operator to start a “client-config-server”, which is a small ESP server running a dummy project. SAS Event Stream Processing Studio uses that ESP server to obtain a list of available connectors, algorithms, and other metadata that it requires. 

An Ingress for each component should also appear in the namespace:

```
   [esp-cloud]$ kubectl -n mudeploy get ingress
   NAME                                               HOSTS            ADDRESS   PORTS     AGE
   espfb                                              sckolo.sas.com             80, 443   25h
M  oauth2-proxy                                       sckolo.sas.com             80, 443   25h
   sas-event-stream-manager-app                       sckolo.sas.com             80, 443   25h
C  sas-event-stream-processing-client-config-server   sckolo.sas.com             80        25h
C  sas-event-stream-processing--app           sckolo.sas.com             80, 443   24h
C  sas-event-stream-processing-streamviewer-app       sckolo.sas.com             80, 443   25h
C  sas-event-stream-processing-studio-app             sckolo.sas.com             80, 443   25h
M  uaa                                                sckolo.sas.com             80, 443   25h
```

**Note to Scott** Amanda reports that client-config-server will show up last. The rest of the pods show up as “running” almost instantaneously  

## Accessing Projects and Servers 

The following URL and context roots are valid to access projects and servers:

```
Project X    --   https://<namespace>.sas.com/SASEventStreamProcessingServer/project/X/eventStreamProcessing/v1/
Metering     --   https://<namespace>.sas.com/SASEventStreamProcessingMetering/eventStreamProcessing/v1/meterData
Studio       --   https://<namespace>.sas.com/SASEventStreamProcessingStudio
Streamviewer --   https://<namespace>.sas.com/SASEventStreamProcessingStreamviewer
ESM          --   https://<namespace>.sas.com/SASEventStreamManager
FileBrowser  --   https://<namespace>.sas.com/files
```

#### Query a Project
Suppose that the Ingress domain root is `sas.com`, the namespace is `esp`, and the project's service name is **array**.  

After deployment, you can query a project deployed in an **open** environment as follows:
```
     curl https://esp.sas.com:80/SASEventStreamProcessingServer/project/array/eventStreamProcessing/v1/
```
You can query a project deployed in a **multi-user** environment as follows:
```
     curl https://esp.sas.com:80/SASEventStreamProcessingServer/project/array/eventStreamProcessing/v1/ \
     -H 'Authorization: Bearer <put a valid access token here>'
```

#### Query the Metering Server
**Note:** You cannot access the metering server through a web browser.

Suppose that the Ingress domain root is `sas.com`, and the namespace is `esp`. 

After deployment, you can perform a simple query of the metering server deployed in an **open** environment as follows:
```
     curl https://esp.sas.com/SASEventStreamProcessingMetering/eventStreamProcessing/v1/meterData
```
You can perform a simple query of the metering server deployed in a **multi-user** environment as follows:
```
     curl https://esp.sas.com/SASEventStreamProcessingMetering/eventStreamProcessing/v1/meterData \
     -H 'Authorization: Bearer <put a valid access token here>'
```


#### Access Graphical Clients
After deployment, you can access graphical clients in an **open** or **multiuser** deployment through the following URLs:
```
Event Stream Processing Studio          -- https://esp.sas.com/SASEventStreamProcessingStudio
Event Stream Processing Streamviewer    -- https://esp.sas.com/SASEventStreamProcessingStreamviewer
Event Stream Processing Manager         -- https://esp.sas.com/SASEventStreamManager
```

## Configuring for Multiple Users

For information about adding service and user accounts and credentials, see [Oauth2](/esp-cloud/oauth2)

## Using filebrowser

The filebrowser application on GitHub enables you to access the persistent store used by the Kubernetes pods.  

The filebrowser application is installed in your Kubernetes cluster automatically for convenience. It can be accessed
through a browser at:

     https://<namespace>.<ingress domain root>/files

With filebrowser, you can perform the following tasks:

* copy input files (CSV, JSON, XML) into the persistent store
* view output files written to the persistent store by running projects
* copy large binary model files for analytics (ASTORE files) to the 
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


