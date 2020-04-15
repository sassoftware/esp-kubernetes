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

For example, setting the following environment variables enable the deployment scripts to pick up the appropriate Docker images:

```shell
export IMAGE_ESPESM="docker.sas.com/pdt/sas-esmapplication:6.2.0-20191029.1572337034992"
export IMAGE_ESPSRV="docker.sas.com/pdt/sas-esp:6.2.0-20191029.1572348916638"
export IMAGE_ESPSTRMVWR="docker.sas.com/pdt/sas-espstreamviewer:6.2.0-20191029.1572339074874"
export IMAGE_ESPSTUDIO="docker.sas.com/pdt/sas-espstudio:6.2.0-20191029.1572338415245"
export IMAGE_METERBILL="docker.sas.com/pdt/sas-espmbs:6.2.0-20191029.1572348254917"
export IMAGE_OPERATOR="docker.sas.com/pdt/sas-espcompop:6.2.0-20191029.1572348554623"
```

## Prerequsities
To deploy the images, you must have a running Kubernetes cluster and a have persistent volume available for use.  Work with your Kubernetes administrator to obtain access to a cluster with a persistent volume.

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
