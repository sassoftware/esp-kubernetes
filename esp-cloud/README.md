## Event Strearm Processing Cloud Ecosystem

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

### Deployment

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
