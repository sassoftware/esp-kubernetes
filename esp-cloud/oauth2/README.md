## OAuth2

### Introduction

This directory contains tools that enable you to run SAS Event Stream Processing in a multi-user environment.

A multi-user deployment deploys the following pods: 
* A Pivotal UAA server
* The SAS OAuth2-proxy

### Building the Pivitol UAA Docker image
Use the following Dockerfile to build the Pivotal UAA Docker image. 
```
FROM tomcat:8-jre8-alpine

ENV CATALINA_OPTS="-Xmx800m"
RUN rm $CATALINA_HOME/webapps/ROOT -r -f
ADD cloudfoundry-identity-uaa-4.30.0.war $CATALINA_HOME/webapps/uaa.war
RUN adduser --uid 1001 -D sas
RUN chown -R sas:sas /usr/local/tomcat
USER 1001

EXPOSE 8080
```
Run the following command on that Dockerfile:
```
docker build . -t <docker_repository>/uaa:4.30.0
```
Obtain the WAR file `cloudfoundry-identity-uaa-4.30.0.war` through a Maven repository of your choice. Put the Dockerfile containing the Pivotal UAA Docker image in the same directory as that WAR file.

### Pivotal UAA Secrets and Management

The **mkdeploy** script creates the file `esp-cloud/oauth2/uaa.yaml`. The first few lines define the username and 
password for the client admin account in UAA. 
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: uaa-secret
type: Opaque
data:
  username: YWRtaW4=
  password: YWRtaW5zZWNyZXQ=
```
The passwords are base64 encoded strings. The default values are **admin** for the username and **adminsecret** for the password. You can change them before deployment. 

After the deployment is successful and the uaa pod has started, you can administer the uaa instance with **uaac**, the UAA command line client.

Also in the `esp-cloud/oauth2/uaa.yaml` is a configMap that contains the entire configuration file for the UAA server. You can customize this configuration before deployment. 

**Note:** The file `uaa.yaml` is a template and should not be used as is. Enter your own private key and certificate.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: uaa-config
data:
    uaa.yml: |-
      issuer:
        uri: http://localhost:8080/uaa
      encryption:
        active_key_label: CHANGE-THIS-KEY
        encryption_keys:
        - label: CHANGE-THIS-KEY
          passphrase: CHANGEME
      uaa:
        # The hostname of the UAA that this login server will connect to
        url: http://localhost:8080/uaa
        token:
          url: http://localhost:8080/uaa/oauth/token
 .
 .
 .
```

### Service and User Accounts

In order to configure a multi-user deployment, you must have access to the Cloud Foundry `uaac` command line tool.
The following configuration examples assume that you can run the `uaac` command line tool from a Docker container:
```shell
$ docker run -ti docker.sas.com/sckolo/esp-test-images/uaac-01:latest /bin/sh
```

The following instructions create the connection to the uaa server, create a application service, and create a user.

Set up the connection to the UAA server:
```
uaac target https://<domain>.com/uaa --skip-ssl-validation
uaac token client get admin -s adminsecret
```

Create a service account:
```
uaac client add sv_client --authorities "uaa.resource"  --scope "openid email"  --autoapprove "openid" --authorized_grant_types "authorization_code refresh_token password client_credentials"  --redirect_uri https://<domain>.com/oauth2 -s secret
```

Create one or more user accounts:
```
uaac user add esp  --emails <name>@<domain>.com -p esppw
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

