## Operator

### Introduction

This directory contains tools that enable you to run ESP in a multiuser environment.

A multiuser deployment will have deployed the following pods: 
* A Pivitol UAA server
* The SAS oauth2-proxy


### Prerequisites

In order to configure a multiuser ESP deployment, access to the Cloud Foundry uaac command line tool is required.
The following configuration examples assumes you can run the `uacc` command line tool from a docker container:
```shell
$ docker run -ti docker.sas.com/sckolo/esp-test-images/uaac-01:latest /bin/sh
```

The following instructions create the connection to the uaa server, create a application service, and create a user.
```
#
# set up connection to uaa server.
#
uaac target https://sckolo.sas.com/uaa --skip-ssl-validation
uaac token client get admin -s adminsecret

#
# create the service account
#
uaac client add sv_client --authorities "uaa.resource"  --scope "openid email"  --autoapprove "openid" --authorized_grant_types "authorization_code refresh_token password client_credentials"  --redirect_uri https://sckolo.sas.com/oauth2 -s secret


#
# add users to UAA server
#
uaac user add esp  --emails scott.kolodzieski@sas.com -p esppw
```

The UAA server persists data to the running postgres database, so is
durable with respect to clusterwide restarts; provided the same
persitent volume is used.
