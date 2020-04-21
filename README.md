## esp-kubernetes

**Note:** These instructions are specific to SAS Event Stream Processing 7.1 or later.

## changes
For changes between releases please vist the [Changelog](CHANGELOG.md)

**Note:** These changes have not been rolled into the documentation yet. all README.cm files on the develop branch still have ESP 6.2 content. 

## Overview
This project is a repository of tools that enable you to develop, deploy, and test an ESP server and SAS Event Stream Processing 
clients in a Kubernetes cluster.  The tools consist of a set of deployment scripts, YAML template files, and sample projects (XML files) that you can run in the cluster. Before you use the tools available in this project, you must download the pre-built Docker images made available through your 
SAS Event Stream Processing Software Order Email (SOE).  

See the 
[SAS Event Stream Processing on Linux: Deployment Guide ](http://pubshelpcenter.unx.sas.com:8080/test/?cdcId=espcdc&cdcVersion=6.2&docsetId=dplyesp0phy0lax&docsetTarget=titlepage.htm&locale=en) for information about how to download the required Docker images and load them onto a local Docker repository. 

The deployment scripts supplied require you to do either of the following:
* Specify the location of the images on the command line
* Set several environmment variables that indicate the location of the images

## Prerequsities
To deploy the images, you must have a running Kubernetes cluster and a have persistent volume available for use.  Work with your Kubernetes administrator to obtain access to a cluster with a persistent volume.

For a multi user deployment, there are a few more prerequsites:
* Access to a Pivitol UAA server in a containor
* Access to the "uaac" Pivitol UAA command line tool to configure the UAA server.

It is easy to create your own UAA Docker container. Download a recent UAA war (such as: cloudfoundry-identity-uaa-4.30.0.war) file from any Maven repository and use the following Dockerfile:

```
FROM tomcat:8-jre8-alpine

ENV CATALINA_OPTS="-Xmx800m"

RUN rm $CATALINA_HOME/webapps/ROOT -r -f
ADD cloudfoundry-identity-uaa-4.30.0.war $CATALINA_HOME/webapps/uaa.war

EXPOSE 8080
```

A convenient way to run the uaac command line client is to build a Docker container containing just the uaac client.
Use the following Dockerfile:
```
FROM ruby:2.6-alpine3.9

# TODO: remove after https://github.com/docker-library/ruby/pull/209 was fixed.
ENV PATH "/usr/local/bundle/bin:${PATH}"

RUN apk add --no-cache musl-dev gcc make g++

RUN gem install cf-uaac -v 3.2.0 --no-document
```

## Getting Started

The entire Event Stream Processing cloud deployment is done from a single directory, esp-cloud. A single script allows 
for the deployment of the basic ESP operator, and optionally the graphical clients. 

The deployment can be done in open mode (no TLS or user authentication), or multi-user mode, which provides full authentication via a UAA server, and comes with TLS enabled by default. 

See: [ESP cloud](/esp-cloud) for further details. 


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
