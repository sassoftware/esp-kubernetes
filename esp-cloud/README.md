## Event Stream Processing Cloud Ecosystem

### Components
[Operator](/esp-cloud/operator) - Contains YAML template files, and projects to deploy the SAS Event Stream Processing metering server and the ESP operator. 

The following Docker images are deployed from this location:
  * SAS Event Stream Processing metering server
  * ESP operator
  * Open source postgres database (this could be easily replaced by any Postgres database)
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
  * Pivitol UAA server (this is configered to store user credential in postgres, but could easily be configured to read user credentials from other IM systems)

Each of these subdirectories contain README files with more specific, detailed instructions.

### Generating a Deployment

It is required to set a number of enviroment variables that point to the docker images that will be used in the deployment. 

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

The mkdeploy script is used to create a set of deployment yaml files. It uses the environment variables mentioned to locate the docker images, and a few passed parameters to specify namespace, ingress root, license, and type of deployment.

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

    
The options `-C` and `-M` are optional, which allows for four types of deployments:

1. **Open deployment (no authentication or TLS)** with no Graphical Clients. 
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt -n sckolo -d sas.com
```
2. **Open deployment (no authentication or TLS)** with all Graphical Clients. 
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt -n sckolo -d sas.com -C
```
3. **Multi-user deployment (UAA authentication and TLS)** with no Graphical Clients.
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt -n sckolo -d sas.com -M
```
4. **Multi-user deployment (UAA authentication and TLS)** with all Graphical Clients.
```shell
[esp-cloud]$ ./bin/mkdeploy -r -l ../../LICENSE/SASViyaV0400_09QTFR_70180938_Linux_x86-64.jwt -n sckolo -d sas.com -C -M
```


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
namespace. The pods(ingress) below marked with a **M** only appear in a Multi-usre deployment. The pods(ingress) marked with a **C** only appear if Graphical Clients are included in the deployment. 

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

### Project and Server access

**Note:** The *-d* (Ingress domain root) parameter specified in the mkdeploy command is used to create Ingress routes for the deployed pods. All Event stream processing components within the kubernetes cluster are now accessed via specific context roots and a single ingress host. The ingress host is of the form `<namespace>.<ingress domain root>`. The following url and context roots are valid:

```
Project X    --   <namespace>.sas.com/SASEventStreamProcessingServer/project/X 
Metering     --   <namespace>.sas.com/SASEventStreamProcessingMetering/SASESP/meterData
Studio       --   <namespace>.sas.com/SASEventStreamProcessingStudio
Streamviewer --   <namespace>.sas.com/SASEventStreamProcessingStreamviewer
ESM          --   <namespace>.sas.com/SASEventStreamManager
FileBrowser  --   <namespace>.sas.com/files
```


#### Metering server
Suppose the ingress domain root is `sas.com`, and the namespace is `esp`. 

A simple query of the metering server deployed in an **open** environment would be:
```
     curl http://esp.sas.com:80/SASEventStreamProcessingMetering/eventStreamProcessing/SASESP/meterData
```
A simple query of the metering server deployed in a **multi-user** environment would be:
```
     curl http://esp.sas.com:80/SASEventStreamProcessingMetering/eventStreamProcessing/SASESP/meterData \
     -H 'Authorization: Bearer <put a valid access token here>'
```

#### ESP Project
Suppose the ingress domain root is `sas.com`, and the namespace is `esp` and the projects service name is **array**.  

A simple query of the project deployed in an **open** environment would be:
```
     curl http://esp.sas.com:80/SASEventStreamProcessingServer/project/array/SASESP
```
A simple query of the project deployed in a **multi-user** environment would be:
```
     curl http://esp.sas.com:80/SASEventStreamProcessingServer/project/arraySASESP \
     -H 'Authorization: Bearer <put a valid access token here>'
```

#### Graphical Clients
The Graphical clients in an **open** deployment can be accessed via these URL's
```
Event Stream Processing Studio          -- http://esp.sas.com/SASEventStreamProcessingStudio
Event Stream Processing Streamviewer    -- http://esp.sas.com/SASEventStreamProcessingStreamviewer
Event STream Processing Manager         -- http://esp.sas.com/SASEventStreamManager
```
The Graphical clients in an **multi-user** deployment can be accessed via these URL's
```
Event Stream Processing Studio          -- https://esp.sas.com/SASEventStreamProcessingStudio
Event Stream Processing Streamviewer    -- https://esp.sas.com/SASEventStreamProcessingStreamviewer
Event STream Processing Manager         -- https://esp.sas.com/SASEventStreamManager
```

### Multi-user configuration

See the dicussion of adding servide / user accounts and credential here: [Oauth2](/esp-cloud/oauth2)

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

