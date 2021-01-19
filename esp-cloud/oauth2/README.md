## OAuth2

### Introduction

This directory contains tools that enable you to run SAS Event Stream Processing in a multi-user environment.

A multi-user deployment deploys the following pods: 
* A Pivotal UAA server
* The SAS OAuth2-proxy

### Service and User Accounts

In order to configure a multi-user deployment, you must have access to the Cloud Foundry `uaac` command line tool. This is automated using a docker image of this tool and the supplied script *uaatool*. **Note:** This script runs a pre-built docker image containing the **uaac** command line tool. In order to function properly, the docker image needs to be able to access the **UAA** server via the ```https://<namespace>.<domain>``` url. If the hostname ```<namespace>.<domain>``` is not publically resolvable via DNS, the following environment variable should be set to set the ```<namespace>.<domain>``` to ```ip-address``` binding in the docker image. 


```export DOCKER_ARGS="--add-host=<namespace>.<domain>:<ipv4-address>"```

The following instructions create the connection to the uaa server, create an application service, and create a user.

Create a service account (this only needs to be once for a deployment):
```
     $ ./bin/uaatool -u <namespace>.<domain> -C <uaa username>:<uaa password>  -c
```

Create a user account:
```
     $ ./bin/uaatool -u <namespace>.<domain> -C <uaa username>:<uaa password>  -a <username>:<email address>:<password> 
```

The ```./bin/uaatool``` can also delete an existing user, and list all existing users. The full usage for the script is:
```
     $ ./bin/uaatool -?
     Usage: ./bin/uaatool
        REQUIRED options for all commands
 
             -u hostname of uaa server (<namespace>.<doamin>)
             -C <uaa username>:<uaa password>
 
        Commands:
 
        create the uaa application client 'sv_client'
             -c
 
        list all users
             -l
 
        add a new user
             -a <username>:<email address>:<password>
 
        delete an existing user
             -d <username>
```

The UAA server persists data to the running PostgreSQL database so that it is
durable with respect to cluster-wide restarts, provided that the same
persitent volume is used.


You need an access token in order to access the metering server or a running ESP server through the curl command. You can query the uaa server for the access token. For example:

```
[esp-cloud]$ curl -k -X POST 'https://opencmdpg.sas.com/uaa/oauth/token' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Accept: application/json' -d 'client_id=sv_client&client_secret=secret&grant_type=password&username=USERNAME&password=PASSWORD'
{"access_token":"eyJhbGciOi<characters>","token_type":"bearer","id_token":"eyJhb<characters>","refresh_token":"eyJhb<characters>","expires_in":<characters>,"scope":"openid","jti":"<characters>"}
```

**Note:** You need only the **access token** in order to use the curl command to access the metering server or SAS Event Stream Processing projects.

