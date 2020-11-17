## OAuth2

### Introduction

This directory contains tools that enable you to run SAS Event Stream Processing in a multi-user environment.

A multi-user deployment deploys the following pods: 
* A Pivotal UAA server
* The SAS OAuth2-proxy

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


An access token is required in order to access to the metering server or to a running ESP server through the curl command. You can query the uaa server for the access token. 

```
[esp-cloud]$ curl -k -X POST 'https://opencmdpg.sas.com/uaa/oauth/token' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Accept: application/json' -d 'client_id=sv_client&client_secret=secret&grant_type=password&username=USERNAME&password=PASSWORD'
{"access_token":"eyJhbGciOiJSUzI1NiIsImprdSI6Imh0dHBzOi8vbG9jYWxob3N0OjgwODAvdWFhL3Rva2VuX2tleXMiLCJraWQiOiJrZXktaWQtMSIsInR5cCI6IkpXVCJ9.eyJqdGkiOiI3MDg2NTFmNWEwNzY0Y2E5OTJlOTI1ZTVhZDkyNjI2OSIsInN1YiI6ImRlY2MzOTdmLWY3ZTQtNDRkMC04ODY5LTQyNDZmMmY2OWRjNCIsInNjb3BlIjpbIm9wZW5pZCJdLCJjbGllbnRfaWQiOiJzdl9jbGllbnQiLCJjaWQiOiJzdl9jbGllbnQiLCJhenAiOiJzdl9jbGllbnQiLCJncmFudF90eXBlIjoicGFzc3dvcmQiLCJ1c2VyX2lkIjoiZGVjYzM5N2YtZjdlNC00NGQwLTg4NjktNDI0NmYyZjY5ZGM0Iiwib3JpZ2luIjoidWFhIiwidXNlcl9uYW1lIjoic2VzcHRlc3QiLCJlbWFpbCI6InlpcWluZy5odWFuZ0BzYXMuY29tIiwiYXV0aF90aW1lIjoxNTg3NTgwMTgxLCJyZXZfc2lnIjoiNTJjMTkwZGQiLCJpYXQiOjE1ODc1ODAxODEsImV4cCI6MTYxODY4NDE4MSwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgwL3VhYS9vYXV0aC90b2tlbiIsInppZCI6InVhYSIsImF1ZCI6WyJvcGVuaWQiLCJzdl9jbGllbnQiXX0.C4SnuLBaDQJAl3o5o9-mk_hW6zqy1KOVfUcS11_eozMwJyCbo-6NRgpatnx_KzowlQbVDHfgeRNenFF0OLirND7LZZD6dfuMUZuqGCOG2V3WbYNQ8X4Q_Ts9jVpQ1FD9TTIskVP_vZr0cObfFHKtBCItSG9JOc3foodE7dX3OJ73kfCvqd157Eo1r6Fa2Aw47MdtvGnWTxtz1Y0xY7xWbIBFSk3YRqWXJDWNi6DlTQGfIdCVuBBpTO53No3ZUUHIJVuRsoKNs6sGxRHEB9syLjQ558HshV1pqapJgK7iPSKpRnrFRls2UbyENHCG-GmYpuE6l_Xu1jLHPfVTR6DXcw","token_type":"bearer","id_token":"eyJhbGciOiJSUzI1NiIsImprdSI6Imh0dHBzOi8vbG9jYWxob3N0OjgwODAvdWFhL3Rva2VuX2tleXMiLCJraWQiOiJrZXktaWQtMSIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJkZWNjMzk3Zi1mN2U0LTQ0ZDAtODg2OS00MjQ2ZjJmNjlkYzQiLCJhdWQiOlsic3ZfY2xpZW50Il0sImlzcyI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA4MC91YWEvb2F1dGgvdG9rZW4iLCJleHAiOjE2MTg2ODQxODEsImlhdCI6MTU4NzU4MDE4MSwiYW1yIjpbInB3ZCJdLCJhenAiOiJzdl9jbGllbnQiLCJzY29wZSI6WyJvcGVuaWQiXSwiZW1haWwiOiJ5aXFpbmcuaHVhbmdAc2FzLmNvbSIsInppZCI6InVhYSIsIm9yaWdpbiI6InVhYSIsImp0aSI6IjcwODY1MWY1YTA3NjRjYTk5MmU5MjVlNWFkOTI2MjY5IiwicHJldmlvdXNfbG9nb25fdGltZSI6MTU4NzU3NTgzNjgyNiwiZW1haWxfdmVyaWZpZWQiOnRydWUsImNsaWVudF9pZCI6InN2X2NsaWVudCIsImNpZCI6InN2X2NsaWVudCIsImdyYW50X3R5cGUiOiJwYXNzd29yZCIsInVzZXJfbmFtZSI6InNlc3B0ZXN0IiwicmV2X3NpZyI6IjUyYzE5MGRkIiwidXNlcl9pZCI6ImRlY2MzOTdmLWY3ZTQtNDRkMC04ODY5LTQyNDZmMmY2OWRjNCIsImF1dGhfdGltZSI6MTU4NzU4MDE4MX0.lNu2H15iHHHhWzLfffXSrUDejblwmMulwKlwPgxDu34DezhO_1AF_bjGrEz91y1Z6gfJB2RHWACU8_aIYTHu4GwrGLf9U_UQem2IK68RUtYg-Zg7-GuYE0cqjtAVccdMFpEdXWLojHAaitvRXFuxOEg20_htXlQuLjPfZ0ZJYVZGHhG6_fOycTBUssB3hUZgAVYPSr9CzUpE530l-dkv82a9rS9z76mDqaJW5SAbsuFS9naq_0JqHtm_AlBPLrs3kNLKnDOZPRHtNwELDoSu5cBqMB-QEm_sRxIpj7IkVRQ6mqfRFGGcgwseDQpxTtU7F1OwPscXA2kEjoWwXP5XpA","refresh_token":"eyJhbGciOiJSUzI1NiIsImprdSI6Imh0dHBzOi8vbG9jYWxob3N0OjgwODAvdWFhL3Rva2VuX2tleXMiLCJraWQiOiJrZXktaWQtMSIsInR5cCI6IkpXVCJ9.eyJqdGkiOiI1MWExMWI3MTc5M2U0YTE0YjVkZTkzZTBjNWYwOWM5NS1yIiwic3ViIjoiZGVjYzM5N2YtZjdlNC00NGQwLTg4NjktNDI0NmYyZjY5ZGM0IiwiaWF0IjoxNTg3NTgwMTgxLCJleHAiOjE2MTg2ODQxODEsImNpZCI6InN2X2NsaWVudCIsImNsaWVudF9pZCI6InN2X2NsaWVudCIsImlzcyI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA4MC91YWEvb2F1dGgvdG9rZW4iLCJ6aWQiOiJ1YWEiLCJhdWQiOlsib3BlbmlkIiwic3ZfY2xpZW50Il0sImdyYW50ZWRfc2NvcGVzIjpbIm9wZW5pZCJdLCJhbXIiOlsicHdkIl0sImF1dGhfdGltZSI6MTU4NzU4MDE4MSwiZ3JhbnRfdHlwZSI6InBhc3N3b3JkIiwidXNlcl9uYW1lIjoic2VzcHRlc3QiLCJvcmlnaW4iOiJ1YWEiLCJ1c2VyX2lkIjoiZGVjYzM5N2YtZjdlNC00NGQwLTg4NjktNDI0NmYyZjY5ZGM0IiwicmV2X3NpZyI6IjUyYzE5MGRkIn0.arlHT-xbitL7imyexpVqGD4kH5F0BwWJl5qbhSUrRaLHDf25N3XPLl_hZfAlXN-EveiQX55skpJeE9QXCCdvZS8C8t-K-H2Djujj4ugGgZnwhAOg4VfyJSxlFOD4gLGinzftcW8o2caC6fU70gsCPO2-vaBrAHpLrbZT68doHp_JHBa26E1KKHNS1PiIvSR8ifz7d9jHYDOUhuu1LNtfSM50qhprlSGWn_dqXyXrh3tQo7D8_9DopWN2XAdcNm1q70Q_mLgeqXAh9PTGsY59pqeZB2EXK77rz9rWnFhRpC9VHpBKxZ7P9ty9yOlOqy1KSwvkcwNZLa4wQ2s2dkDvAw","expires_in":31103999,"scope":"openid","jti":"708651f5a0764ca992e925e5ad926269"}
```

**Note:** This query returns a lot of information. You need only the **access token** to use the curl command to access the metering server of SAS Event Stream Processing projects.

