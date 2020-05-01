## esp-kubernetes

**Note:** These instructions are specific to SAS Event Stream Processing 7.1 or later.

## Changes
For changes between releases, please read the [changelog](CHANGELOG.md).

## Overview
This project is a repository of tools that enable you to develop, deploy, and test an ESP server and SAS Event Stream Processing 
clients in a Kubernetes cluster.  These tools are applicable to the following deployment types:
* Lightweight open multi-user, multi-tenant deployment
* Lightweight open, single-user deployment

The tools consist of a set of deployment scripts, YAML template files, and sample projects (XML files) that you can run within the cluster. Before you use the tools available in this project, you must download the pre-built Docker images made available through your 
SAS Event Stream Processing Software Order Email (SOE).  The SOE guides you to information about how to download the required Docker images and load them onto a local Docker repository.

## Components of the SAS Event Stream Processing Cloud Ecosystem

[Operator](/esp-cloud/operator) - contains YAML template files, and projects to deploy the SAS Event Stream Processing metering server and the ESP operator. 

The following Docker images are deployed from this location:
  * SAS Event Stream Processing metering server
  * ESP operator
  * Open source postgres database (this could be easily replaced by any Postgres database)
  * Open source filebrowser to manage the persistent volume


[Clients](/esp-cloud/clients) - contains YAML template files, and projects to deploy SAS Event Stream Processing 
graphics clients.  

The following Docker images are deployed from this location: 
  * SAS Event Stream Processing Studio
  * SAS Event Stream Processing Streamviewer
  * SAS Event Stream Manager

[Oauth2](/esp-cloud/oauth2) - contains YAML template files for supporting multi-user installs.

The following Docker images are deployed from this location: 
  * SAS Oauth2_proxy
  * Pivitol UAA server (this is configered to store user credential in postgres, but could easily be configured to read user credentials from other IM systems)

Each of these subdirectories contain README files with more specific, detailed instructions.

## Prerequsities

### Persistent Volume
To deploy the images, you must have a running Kubernetes cluster and a have persistent volume available for use.  Work with your Kubernetes administrator to obtain access to a cluster with a persistent volume.

To deploy the SAS Event Stream Processing cloud ecosystem, you must have a running Kubernetes cluster and a persistent volume. The persistent volume is used to store the following:

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
         storage: 20Gi  # volume size requested
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

### Additional Prerequisities for a Multi-user Deployment
For a multi-user deployment, there the following additional prerequsites:
* Access to a Pivitol UAA server in a container
* Access to the "uaac" Pivitol UAA command line tool to configure the UAA server.

To create your own UAA Docker container, download a recent UAA war (such as: cloudfoundry-identity-uaa-4.30.0.war) file from any Maven repository and use the following Dockerfile:

```
FROM tomcat:8-jre8-alpine

ENV CATALINA_OPTS="-Xmx800m"

RUN rm $CATALINA_HOME/webapps/ROOT -r -f
ADD cloudfoundry-identity-uaa-4.30.0.war $CATALINA_HOME/webapps/uaa.war

EXPOSE 8080
```
A convenient way to run the uaac command line client is to build a Docker container with just the uaac client.
Use the following Dockerfile:
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
IMAGE_UAA         = "name of image for Pivitol UAA Server"
```
Alternatively, you can specify the specific location of the images on the command line.

Perform the SAS Event Stream Processing cloud deployment from a single directory, esp-cloud. A single script enables the deployment of the basic ESP operator and the graphical clients. 

The deployment can be done in open mode (no TLS or user authentication), or multi-user mode, which provides full authentication through a UAA server. Multi-uesr mode has TLS enabled by default. 

See: [ESP cloud](/esp-cloud) for further details. 

### Generate a Deployment

Use the mkdeploy script to create a set of deployment YAML files. The script uses the environment variables you set to locate the Docker images and pass parameters to specify namespace, ingress root, license, and type of deployment.

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

    
The options `-C` and `-M` are optional, which generate the following deployments:

1. **Open deployment (no authentication or TLS)** with no graphical clients. 
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt \
                            -n sckolo -d sas.com
```
2. **Open deployment (no authentication or TLS)** with all graphical clients. 
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt \
                            -n sckolo -d sas.com -C
```
3. **Multi-user deployment (UAA authentication and TLS)** with no Graphical Clients.
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt \
                            -n sckolo -d sas.com -M
```
4. **Multi-user deployment (UAA authentication and TLS)** with all Graphical Clients.
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
These are base64 encoded strings. The defualt values are **esp** for the username and **esp_in_cloud** for the password. Thay can be adjusted prior to deployment. 

After the deployment is successful and the Postgres pod has started, the Postgres instance can be administered with **psql** from the Kubernetes cluster. Use the following command to connect to psql: 

```shell
kubectl -n sckolo exec -it postgres-deployment-6f9d6cc8cc-mhx79 -- psql -U esp
```

**Note:** the name of your postgres pod will differ from **postgres-deployment-6f9d6cc8cc-mhx79**. 


### Deploy Images in Kubernetes

After you have revised the manifest files that reside in deploy/, deploy images on the Kubernetes
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

After the deployment is completed you should see active pods in your
namespace. The pods(ingress) below marked with a **M** only appear in a Multi-user deployment. The pods(ingress) marked with a **C** only appear when graphical clients are included in the deployment. 

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
An Ingress for the each compenent should also appear in the namespace:

```
   [esp-cloud]$ kubectl -n mudeploy get ingress
   NAME                                               HOSTS            ADDRESS   PORTS     AGE
   espfb                                              sckolo.sas.com             80, 443   25h
M  oauth2-proxy                                       sckolo.sas.com             80, 443   25h
   sas-event-stream-manager-app                       sckolo.sas.com             80, 443   25h
C  sas-event-stream-processing-client-config-server   sckolo.sas.com             80        25h
C  sas-event-stream-processing-metering-app           sckolo.sas.com             80, 443   24h
C  sas-event-stream-processing-streamviewer-app       sckolo.sas.com             80, 443   25h
C  sas-event-stream-processing-studio-app             sckolo.sas.com             80, 443   25h
M  uaa                                                sckolo.sas.com             80, 443   25h
```

## Accessing Projects and Servers 

**Note:** Use the *-d* (Ingress domain root) parameter specified in the mkdeploy command to create Ingress routes for the deployed pods. All SAS Event Stream Processing applications within the kubernetes cluster are now accessed through specific context roots and a single Ingress host. The Ingress host is of the form `<namespace>.<ingress domain root>`. The following URL and context roots are valid:

```
Project X    --   <namespace>.sas.com/SASEventStreamProcessingServer/project/X 
Metering     --   <namespace>.sas.com/SASEventStreamProcessingMetering/SASESP/meterData
Studio       --   <namespace>.sas.com/SASEventStreamProcessingStudio
Streamviewer --   <namespace>.sas.com/SASEventStreamProcessingStreamviewer
ESM          --   <namespace>.sas.com/SASEventStreamManager
FileBrowser  --   <namespace>.sas.com/files
```


#### Querying the Metering Server
Suppose that the ingress domain root is `sas.com`, and the namespace is `esp`. 

You can perform a simple query of the metering server deployed in an **open** environment as follows:
```
     curl http://esp.sas.com:80/SASEventStreamProcessingMetering/eventStreamProcessing/SASESP/meterData
```
You can perform a simple query of the metering server deployed in a **multi-user** environment as follows:
```
     curl http://esp.sas.com:80/SASEventStreamProcessingMetering/eventStreamProcessing/SASESP/meterData \
     -H 'Authorization: Bearer <put a valid access token here>'
```

### Querying a Project
Suppose that the Ingress domain root is `sas.com`, and the namespace is `esp` and the projects service name is **array**.  

You can query a project deployed in an **open** environment as follows:
```
     curl http://esp.sas.com:80/SASEventStreamProcessingServer/project/array/SASESP
```
You an query a project deployed in a **multi-user** environment as follows:
```
     curl http://esp.sas.com:80/SASEventStreamProcessingServer/project/arraySASESP \
     -H 'Authorization: Bearer <put a valid access token here>'
```

### Accessing Graphical Clients
You can access graphical clients in an **open** deployment through the following URLs:
```
Event Stream Processing Studio          -- http://esp.sas.com/SASEventStreamProcessingStudio
Event Stream Processing Streamviewer    -- http://esp.sas.com/SASEventStreamProcessingStreamviewer
Event STream Processing Manager         -- http://esp.sas.com/SASEventStreamManager
```
You can access graphical clients in an **multi-user** deployment through the following URLs:
```
Event Stream Processing Studio          -- https://esp.sas.com/SASEventStreamProcessingStudio
Event Stream Processing Streamviewer    -- https://esp.sas.com/SASEventStreamProcessingStreamviewer
Event STream Processing Manager         -- https://esp.sas.com/SASEventStreamManager
```

## Configuring for Multiple Users

See the dicussion of adding service and/or user accounts and credentials here: [Oauth2](/esp-cloud/oauth2)

## Using filebrowser

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

## Contributing

We welcome your contributions! Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on how to submit contributions to this project.

## License

This project is licensed under the [Apache 2.0 License](LICENSE).

## Additional Resources

The [SAS Event Stream Processing product support page](https://support.sas.com/en/software/event-stream-processing-support.html)
contains:
* Current and past product documentation
* Instructional videos
* Examples
* Training courses
* Featured blogs
* Featured community topics


