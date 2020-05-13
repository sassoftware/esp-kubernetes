## Web-based Clients

### Introduction

This directory contains the deployment YAML template files, used to deploy the SAS Event Stream Processing web-based clients:

* SAS Event Stream Processing Studio
* SAS Event Stream Processing Streamviewer
* SAS Event Stream Manager

### Example

**Note:** This example is based on the ESP operator example in the  [Read and write to the persistent volume](../operator#read-and-write-to-the-persistent-volume) section of the ESP operator README. As a result, some of the steps might have been completed already.

You need the following files to run this example: 

* esp-cloud/deploy/examples/input/array_input01.csv.gz
* esp-cloud/deploy/examples/example-1.xml. 

To run the example:
1. Uncompress the input file deploy/examples/input/array_input01.csv.gz. 
1. Copy the expanded file (array_input01.csv) to the directory input/ on your persistent volume. You can do this with the filebrowser described in the ESP operator README (for more information, see [filebrowser](/../../#using-filebrowser)) or with any standard Unix or Kubernetes tools.
1. Upload the esp-cloud/deploy/examples/example-1.xml file to SAS Event Stream Processing Studio or SAS Event Stream Manager.
1. Test the project using the test mode in SAS Event Stream Processing Studio, or deploy the project using SAS Event Stream Processing Studio or SAS Event Stream Manager.
1. When the project has been loaded, use SAS Event Stream Processing Streamviewer to subscribe to the windows for the running project.

For more information, see the user documentation for SAS Event Stream Processing Studio, SAS Event Stream Processing Streamviewer, and SAS Event Stream Manager.
