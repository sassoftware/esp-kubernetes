## Single-User Clients

### Introduction

This directory contains tools that enable you to develop, deploy, and test client tools that run alongside an open ESP server in a Kubernetes cluster.
The resources consist of a set of scripts, YAML template files, and sample projects (XML files) that you can run on the ESP server.

With these tools, you can deploy the following clients:
* SAS Event Stream Processing Studio
* SAS Event Stream Processing Streamviewer
* SAS Event Stream Manager

The deployment is an open deployment (it does not include authentication) and is meant to be used by a single user or a small cooperative team. 

### Prerequisites

Before running the script to deploy the clients, you must have already deployed the ESP operator. For more information, see [Operator](/operator).

### Create a Deployment

The YAML file templates that are located in the single_user_clients/templates/ directory cannot be deployed immediately. They contain placeholder values that must 
be replaced with values associated with a specific Kubernetes cluster. The bin/mkdeploy script enables you to substitute real values for placeholder values.

**Note:** Before running the bin/mkdeploy script, you must change directory to single_user_clients.

The options for this script are:

    [operator]$ ./bin/mkdeploy
    Usage: ./bin/mkdeploy
     GENERAL options

          -r                          -- remove existing deploy/
                                          before creating
          -y                          -- no prompt, just execute
          -n <namespace>              -- specify K8 namespace
          -d <ingress domain root>    -- project domain root,
                                          proj.ns.<domain root>
          -l <esp license file>       -- SAS ESP license

     options for operator deployment

          -o <esp operator image>     -- esp operator docker image
          -s <esp server image>       -- esp server docker image
          -m <esp meter image>        -- esp metering docker image
          -a <sas meter agent image>  -- esp meter agent docker image

     options for single user client deployment

          -e <esm image>              -- esp esm docker image
          -t <esp studio image>       -- esp studio docker image
          -v <esp streamviewer image> -- esp streamviewer docker image

The usage information above shows two optional sections, one for `operator deployment` and the other for `single user client deployment`. The options for 
`single user client deployment` are relevant for deploying the clients.

**Note:** The *-l* option requires a TXT license file, not a JWT file.

Here is a sample invocation of the ..bin/mkdeploy script from the single_user_clients directory:

    [single_user_clients]$ ../bin/mkdeploy -r -n cmdline 
        -e docker.sas.com/pdt/sas-esmapplication:6.2.0-20191024.1571905032728 
        -t docker.sas.com/pdt/sas-espstudio:6.2.0-20-191024.1571906412613  
        -v docker.sas.com/pdt/sas-espstreamviewer:6.2.0-20191024.1571907074722
        -d sas.com
        -l ~/license/SASViyaV0300_09PSJN_Linux_x86-64.txt
        
This invocation performs parameter substitutions and produces a set of deployable manifests in the single_user_clients/deploy/ directory.

The license file is required by SAS Event Stream Manager. 

**Note:** The *-d* (Ingress domain root) parameter and the *-n* (namespace) is used to create Ingress routes for all three client 
applications:
- `espstudio.<namespace>.<domain>`
- `esm.<namespace>.<domain>`
- `streamviewer.<namespace>.<domain>`

The full URI to the application includes the application context. For example: `SASEventStreamProcessingStudio`. Examples are included later in this README file. 

### Persistent Volume

**Note:** The following instructions assume that you are already familiar with the [Persistent Volume](/operator#persistent-volume) section in the ESP operator README.

A persistent volume is required to store H2 database files, and input and output files used to test ESP projects.

As described in [Persistent Volume](/operator#persistent-volume), three directories are created and are accessible from all three clients. The paths to these directories include the variable `<namespace>`. The paths are:

- `/mnt/data/<namespace>/DB`
- `/mnt/data/<namespace>/input`
- `/mnt/data/<namespace>/output`

**Note:** When testing SAS Event Stream Processing projects that require persistent storage for input and ouptut data, the correct file paths must be used in the XML code. 
The example project described later in this README is already configured with the correct paths.

### Deploy in Kubernetes

After you have revised the manifest files that reside in the single_user_clients/deploy/ directory, deploy them on the Kubernetes
cluster with the script bin/dodeploy.

**Note:** Before running the bin/mkdeploy script, you must change directory to single_user_clients.

    Usage: ../bin/dodeploy
         -n <namespace>             -- specify K8 namespace

**Note:** The namespace already exists because you deployed the ESP operator earlier. For more information, see [Operator](/operator). If you
have not deployed the ESP operator yet, you must deploy it now. 

After the deployment is completed you see three new active pods in your namespace. For example:

    [single_user_clients]$ kubectl -n esp get pods
    NAME                                       READY   STATUS    RESTARTS   AGE
    esm-deployment-6f876cc6fc-gvv4w            1/1     Running   0          1m
    espstudio-deployment-5f77cdb484-m9bd9      1/1     Running   0          1m
    streamviewer-deployment-5d88c657d4-sct96   1/1     Running   0          1m

    [single_user_clients]$ kubectl -n esp get ingress
    NAME           HOSTS                                          ADDRESS   PORTS   AGE
    esm            esm.examplenamespace.exampledomain                       80      1m
    espstudio      espstudio.examplenamespace.exampledomain                 80      1m
    streamviewer   streamviewer.examplenamespace.exampledomain              80      1m
    
The clients take approximately two minutes to start. To check for progress, you can tail the logs of the pods. For example:

    [single_user_clients]$ kubectl -n mynamespace logs -f esm-deployment-6f876cc6fc-gvv4w 
    
When the clients have started, you can access each client with a URL that consists of the host name (as defined in the Ingress) and the application context name. For example, using the namespace `examplenamespace` and domain `exampledomain`, the URLs are:

- `http://espstudio.examplenamespace.exampledomain/SASEventStreamProcessingStudio`
- `http://esm.examplenamespace.exampledomain/SASEventStreamManager`
- `http://streamviewer.examplenamespace.exampledomain/SASEventStreamProcessingStreamviewer`

### Example

**Note:** This example is based on the ESP operator example in the  [Read and write to the persistent volume](/operator#read-and-write-to-the-persistent-volume) section of the ESP operator README. As a result, some of the steps might have been completed already.

You need the following files to run this example: 

* operator/deploy/examples/input/array_input01.csv.gz
* operator/deploy/examples/example-1.xml. 

To run the example:
1. Uncompress the input file deploy/examples/input/array_input01.csv.gz. 
1. Copy the expanded file (array_input01.csv) to the directory <namespace>/input on your persistent volume. You can do this with the filebrowser described in the ESP operator README (for more information, see [filebrowser](/sckolo/esp-kubernetes/tree/master/operator#filebrowser)) or with any standard Unix or Kubernetes tools.
1. Upload the operator/deploy/examples/example-1.xml file to SAS Event Stream Processing Studio or SAS Event Stream Manager.
1. Test the project using the test mode in SAS Event Stream Processing Studio, or deploy the project using SAS Event Stream Processing Studio or SAS Event Stream Manager.
1. When the project has been loaded, use SAS Event Stream Processing Streamviewer to subscribe to the windows for the running project.

For more information, see the user documentation for SAS Event Stream Processing Studio, SAS Event Stream Processing Streamviewer, and SAS Event Stream Manager.

### Troubleshooting: Time-out Period

When SAS Event Stream Processing Studio or SAS Event Stream Manager makes a request to the Kubernetes cluster to run a SAS Event Stream Processing project, there can be noticeable delay in completing the task. This can happen when CPU or memory is approaching maximum utilization across the nodes in the cluster. 

You can use the following `kubectl` command to view CPU and memory use across nodes:
`kubectl -n <namespace> top nodes`

In order to handle such delays, the client applications have time-out periods. When the time-out period is reached, the application assumes that the operation has failed and reports an error. However, an error might not have occurred on the cluster: instead, processing the request might be taking longer than expected.

#### Refresh the View

When a time-out occurs, a window with an error message is displayed in the client application. If an error message is not displayed but the client application is not displaying the running project either, refresh the client application: 

- In SAS Event Stream Processing Studio, on the **ESP Servers** page, click ![Refresh list](images/button-refresh.png "Refresh list") on the toolbar. 
- In SAS Event Stream Manager, on the page for the specific deployment, click ![Refresh list](images/button-refresh.png "Refresh list") on the toolbar.

#### Terminate Projects That Are in an Inconsistent State 

In the test mode in SAS Event Stream Processing Studio, if the project fails to load and an error is reported, the cluster might have reported an error or the cluster might still be processing the request. In both cases, you can remove the project from the cluster. Complete the following steps:

1.  In SAS Event Stream Processing Studio, open the **ESP Servers** page.
2.  Click ![Refresh list](images/button-refresh.png "Refresh list") to refresh the view.
3.  Select the ESP server that you want to delete.
4.  Click ![Delete ESP server](images/button-delete.png "Delete ESP server"). The Remove ESP Server window appears.
5.  Click **Yes** to confirm the deletion.

In SAS Event Stream Manager, if the project fails to load and an error is reported, complete the following steps:

1.  In SAS Event Stream Manager, open the page for the specific deployment.
2.  Click ![Refresh list](images/button-refresh.png "Refresh list") to refresh the view. The running ESP server should be displayed.
2.  Click ![Start or stop project in cluster](images/button-cloud.png "Start or stop project in cluster") and select **Stop project and delete server from cluster**. The Stop Project and Delete ESP Server from Cluster window appears.
3.  Click **Delete**.

#### Change the Time-out Period

If a client application has timed out while waiting for the cluster to respond, the cluster might still be processing the request. If this happens, remove the project and try again.  If time-outs are occurring frequently, you can set environment variables to change the duration of the time-outs.

|Client application|Environment variable|Default value|Comment|
|------------------|--------------------|-------------|-------|
|SAS Event Stream Processing Studio|sas_esp_common_kubernetes_timeBeforeAssumingCreationFailure|60|SAS Event Stream Processing Studio requests that a project is loaded on the cluster. After the time-out period is reached, SAS Event Stream Processing Studio reports an error.|
|SAS Event Stream Manager|sas_esp_common_kubernetes_timeBeforeAssumingCreationFailure|60|SAS Event Stream Manager requests that a project is loaded on the cluster. After the time-out period is reached, SAS Event Stream Manager requests reports an error. If set to `0`, there is no time-out.|
|SAS Event Stream Manager|sas_esp_common_kubernetes_timeBeforeAssumingDeletionFailure|20|SAS Event Stream Manager requests that a project is unloaded from the cluster. After the time-out period is reached, SAS Event Stream Manager requests reports an error. If set to `0`, there is no time-out.|

To set the environment variable, edit the `esptudio.yml` file, the `esm.yaml` file, or both. These files are stored in sub-directories of the `single_user_clients/deploy` directory.

You must then redeploy the client applications:
- The remove the client application, use the `kubectl -n <namespace> delete -f <yamlfile>` command.
- To deploy the client application, use the `kubectl -n <namespace> apply -f <yamlfile>` command.

### Troubleshooting: Errors in the Cluster 

If deploying projects from the client applications is failing, you can use `kubectl` to access the Kubernetes logs. Use the `describe` command to view details of the pods and the custom resource `ESPServer`. 

For example: `kubectl -n <namespace> describe ESPServer <espservername>` 

#### Understanding the Naming of ESP Servers

When a SAS Event Stream Processing project is deployed to a Kubernetes cluster and an ESP server is created, the ESP server name does not resemble the project name. Being aware of this is important when you are viewing resources in Kubernetes.

To run a SAS Event Stream Processing project, the client applications must provide a Kubernetes resource name. This resource name forms a part of the URI of the pod on which the ESP server is running. The format of the resource name is very limited. The characters allowed in names are: digits (0-9), lower case letters (a-z), `-`, and `.`. For more information, see https://kubernetes.io/docs/concepts/overview/working-with-objects/names/.

To work around this limitation, the client applications create a hash of the project name to use as the resource name. As a result, you see host names such as `xf1f713c9e000f5d3f280adbd124df4f5.mynamespace.sas.com`. If you query the cluster namespace with a tool such as `kubectl`, you see the hash value (such as `xf1f713c9e000f5d3f280adbd124df4f5`) used in the naming of pods, the service, and the ESP server custom resources. 

#### Understanding an ESP Server Annotated as `configserver`

When troubleshooting errors in the cluster, you might come across an ESP server that is annotated as `configserver`. When you are troubleshooting errors, it is important that you understand the purpose of such an ESP server, so that you do not unintentionally terminate it.

When a project is edited in SAS Event Stream Processing Studio, the application makes a request to a running ESP server to fetch information about connectors and algorithms. This information is used to populate panes and windows in the SAS Event Stream Processing user interface. In a Kubernetes environment, a running ESP server might not exist. SAS Event Stream Processing Studio starts an ESP server for this purpose. 

This ESP server is hidden from the SAS Event Stream Processing Studio user interface, but can be viewed using a tool such as `kubectl`. If you use the `describe` command to view details of the `ESPServer` custom resource started for this purpose, you can observe that the `Annotation` attribute of the custom resource contains the text `configserver`.

#### Understanding How Projects are Displayed in Client Applications

When troubleshooting errors in the cluster, you might notice that the set of SAS Event Stream Processing projects that is displayed in SAS Event Stream Processing Studio is different from the set that is displayed in SAS Event Stream Manager.

SAS Event Stream Processing Studio displays all SAS Event Stream Processing projects that are started in the Kubernetes cluster. These are displayed on the **ESP Servers** page in SAS Event Stream Processing Studio. 

SAS Event Stream Manager displays only SAS Event Stream Processing projects that have been started from a SAS Event Stream Manager deployment. SAS Event Stream Manager organizes ESP servers by deployment and this pattern is also followed in the Kubernetes environment.

