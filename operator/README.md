## Operator

### Introduction

This directory contains tools that enable you to develop, deploy, and test an open ESP server in a Kubernetes cluster. The tools consist of a set of scripts, YAML template files, and sample projects (XML files) that you can run in the ESP server.

The following scripts, which you run from the `operator` directory, are provided:
* bin/mkdeploy - reads the specified manifest located in `templates/`, applies parameters specified on the command line, produces a manifest ready for deployment, and places that manifest in `deploy/`
* bin/dodeploy - deploys the manifest files that are located in `deploy/`
* bin/mkproject - converts sample models into custom resources used by Kubernetes 

With these scripts, you can deploy the following:

* The ESP metering server
* The ESP operator

The ESP operator watches for custom resources that the ESP server can run. When the ESP operator detects a new custom resource, it reads the resource and spins up a single Kubernetes pod running an ESP Server.  The ESP server runs the project that is 
embedded in the custom resource. Further, the ESP operator configures the Ingress endpoint that exposes the REST service for the ESP server.  The ESP server running in the Kubernetes pod uses this endpoint
to communicate with the outside world.

### Prerequisites

To deploy the single-user ESP server, you must have a running Kubernetes cluster and a persistent volume. The persistent volume is used to store the following:

1. The H2 database used by the metering server
2. Any files (csv/xml/json) referenced by the model running on the ESP server
 
### Creating a Deployment

The YAML file templates that are located in the templates/ directory
cannot be deployed immediately. They contain placeholder values that you must replace with values that are associated with your Kubernetes cluster. The bin/mkdeploy script enables you to specify real values to replace the placeholder values.
The options for this script are as follows:

```shell
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
          -c <SAS certificate file>   -- SAS_CA_Certificate.pem

     options for operator deployment

          -o <esp operator image>     -- esp operator docker image
          -s <esp server image>       -- esp server docker image
          -m <esp meter image>        -- esp metering docker image
          -a <sas meter agent image>  -- esp meter agent docker image

     options for single user client deployment

          -e <esm image>              -- esp esm docker image
          -t <esp studio image>       -- esp studio docker image
          -v <esp streamviewer image> -- esp streamviewer docker image
```

Here is a sample invocation of bin/mkdeploy:

```shell
        [~]$ ./bin/mkdeploy -r -n cmdline 
            -o docker.sas.com/shhuan/esp-operator:latest 
            -s docker.sas.com/pdt/sas-esp:6.2.0-20190918.1568806516268  
            -m docker.sas.com/pdt/sas-espmbs:6.2.0-20190916.1568633055007
	    -a repulpmaster.unx.sas.com/lookaside/18b072c9-60bc-45ca-a854-43c3ae8c13b7:latest
            -d sas.com
	    -l ./license/SASViyaV0300_09PTGX_70180938_Linux_x86-64.jwt
	    -c ./license/SAS_CA_Certificate.pem	    
```

This invocation performs parameter substitutions and produces a set of
deployable manifests in the deploy/ directory. 

**Note:** The *-d* (Ingress domain root) parameter is used to create Ingress routes for the
metering server and the ESP server pods that are created by the adapter. For
example, if the domain is sas.com, then the metering server exposes itself
on the Ingress host as `espmeter.<namespace>.sas.com`. For pods that are created by the ESP operator, 
each pod created has a specified
service name.  The ESP operator exposes the REST port of the
pod on `<service name>.<namespace>.sas.com`. Examples of
this appear later in this README file.

### Persistent Volume

After you run ./bin/mkdeploy script and usable deployment manifests are
generated, examine the YAML template file named deploy/pvc.yaml.

```yaml
  #
  # This is the esp-pv claim that esp component pods use.
  #
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
     name: esp-pv
     namespace: esp
  spec:
     accessModes:
       - ReadWriteMany # esp, studio and streamviewer can all write to this space
     resources:
       requests:
         storage: 5Gi  # volume size requested
```

This file specifies the *PersistentVolumeClaim* that the ESP metering server and the ESP
server pods make in Kubernetes. 

**Important**: The system administrator must have already set up a persistent volume that can bind to this claim.

In general, the processes associated with the ESP server run user:**sas**, group:**sas**. Commonly, 
this is associated with uid:**1001**, gid:**1001**. These values are relevant in
the manifest only when starting the metering server. 

In the YAML template file deploy/meter.yaml. the relevant section is as follows:

```yaml
         initContainers:
         - name: config-data
           image: busybox
           #
           # Our nfs PV is owned by sas:sas which is 1001:1001, so
           #    use those credentials to make <namespace>/{DB.input,output}
           #    directories.
           #
           securityContext:
             runAsUser:  1001
             runAsGroup: 1001
           command: ['sh', '-c', 'mkdir -p /mnt/data/cmdline/DB ; mkdir -p /mnt/data/cmdline/input ; mkdir -p /mnt/data/cmdline/output']
           volumeMounts:
           - mountPath: /mnt/data
             name: data
```

This specifies an initialization container that runs prior to starting the
ESP metering server. Its purpose is to create the following three directories:

    DB/
    input/
    output/

These directories are used by the deployment. The metering server creates its persistent
H2 database in DB/, while the input/ and output/ directories are created
for use by running event stream processing projects.

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

After the deployment is completed you should see two active pods in your
namespace:


    [cli]$ kubectl -n cmdline get pods
    NAME                                             READY   STATUS    RESTARTS   AGE
    esp-operator-786f89958d-m2sbr                    1/1     Running   0          19h
    espmeter-deployment-57cc56b5bc-nh8k2             1/1     Running   0          18h

An Ingress for the ESP metering server should also appear in the namespace:

    [cli]$ kubectl -n cmdline get ingress
    NAME       HOSTS                      ADDRESS   PORTS   AGE
    metering   espmeter.cmdline.sas.com             80      23h


### Using filebrowser

filebrowser is a middleware or standalone app that is available on GitHub.
You can use the filebrowser available with these tools to
access the persistent store used by the Kubernetes pods.  

To download the app, go to [filebrowser home page](https://hub.docker.com/r/filebrowser/filebrowser).

With filebrowser, you can perform the following tasks:

* Copy input files (csv, json, xml) into the persistent store
* View output files written to the persistent store by running projects
* Copy large binary model files for analytics (ASTORE files) to the 
persistent store
* Back up the H2 database file to which the ESP metering server writes.

Sample manifests to deploy filebrowser are located in the deploy
directory:

     deploy/fileb.yaml        -- deploy the filebrowser pod
     deploy/ingres-fileb.yaml -- create an ingress for the filebrowser

The Ingress created for the filebrowser is: `espfb.<namespace>.sas.com`. The file browser
is fixed with the root of its filesystem to `<namespace>/` so that the
deployed filebrowser cannot access and data on the persistent store that
might belong to a similar deployment in another namespace. 

When asked for a username/password to access the filebrowser, use `admin/admin`.

For more information, see [filebroswer.xyz](https://filebrowser.xyz/).


### Examples

#### Read and Write to the Persistent Volume

You need the following files to run this example: 

* deploy/examples/input/array_input01.csv.gz
* deploy/examples/example-1.xml

First, uncompress the input file, deploy/examples/input/array_input01.csv.gz.
Then copy the expanded file (array_input01.csv) to the directory `<namespace>/input` on your persistent volume. You can do this with
the filebrowser described in the previous section, or with any standard Unix or Kubernetes tools.

Next, use the bin/mkproject script to convert the
standard ESP XML project (deploy/examples/example-1.xml) into a custom resource.  When this resource is
applied in the Kubernetes cluster, it triggers the ESP operator to start an ESP
server pod, running the specified project. 

The syntax is as follows

```shell
       [~]$ ./bin/mkproject  -x deploy/examples/example-1.xml -s array -o cr_array.yaml

* The -x argument specifes the ESP XML project files
* The -o argument specifies the file to which to write the custom resource
* The -s `<service name>` argument specifies the service name portion of ingress 
  `<service name>.<namespace>.<domain root>` created by the ESP operator.
```

Now apply the custom resource to your Kubernetes cluster:

       [~]$ kubectl -n <namespace> apply -f cr_array.yaml

Verify that the pod is running with the following command:

       [~]$ kubectl -n <namespace> get pods

       NAME                                   READY   STATUS    RESTARTS   AGE
       array-68748c7796-fxpvx                 1/1     Running   0          19m
       esp-operator-588d7fdfd8-hwpm8          1/1     Running   0          19m
       espfb-deployment-7484d75c58-tx8ch      1/1     Running   0          29m
       espmeter-deployment-58bc86b8ff-jtpnm   1/1     Running   0          29m

Use the filebrowser to inspect the directory `<namespace>/output`. You should see
the output csv file (array_output01.csv) that the project has created.

#### Using a Multi-Partitioned Kafka Topic for Autoscaling

The following example shows how to use Kafka with a multi-partitioned topic.
You feed a project data through Kafka.  Kubernetes and the ESP operator scale
the number of project pods according to resource load.

Specifically, you deploy a `strimzi kafka operator` to the cluster and create a three
broker Kafka cluster with a single topic named *sjkinput* spread over 10
partitions. Detailed configuration instructions are beyond the scope of
this document.  Use the following helpful hints:
* Install the Kafka operator per instructions at [Kafka operator](https://operatorhub.io/operator/strimzi-kafka-operator)
* Apply the following two YAML files to your namespace to create the kafka cluster, and then to instantiate the topic on the cluster.
    
     

[kafka]$ cat create-cluster.yaml
```yaml
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
metadata:
  name: kafka-cluster
spec:
  kafka:
    version: 2.2.1
    replicas: 3
    listeners:
      plain: {}
      tls: {}
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      log.message.format.version: '2.2'
    storage:
      type: ephemeral
  zookeeper:
    replicas: 3
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

[kafka]$ cat create-topic.yaml
```yaml
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaTopic
metadata:
  name: sjkinput
  labels:
    strimzi.io/cluster: kafka-cluster
spec:
  partitions: 10
  replicas: 1
  config:
    retention.ms: 600000
    segment.bytes: 1073741824
```

This example runs two ESP models.  
* The first model reads from Kafka and then
computes some values.  The model scales up or down automatically.
* The second model reads a CSV file, loops and loads significant amounts of
data onto Kafka, which forces the first model to consume CPU, and then triggers the
autoscaling of the ESP project.

First uncompress the input file, deploy/examples/input/kafka_input01.csv.gz.
Then copy the
expanded file (kafka_input01.csv) to the directory `<namespace>/input` on your
persistent volume.

Next you create deployable custom resources for the two models. First, create the model that reads from a Kafka topic and autoscales.

       [cli]$ ./bin/mkproject  -x deploy/examples/example-2.xml -s source -o cr_source.yaml

Then create the model that reads from a CSV file on the persistent volume and pushes messages to
Kafka:

       [cli]$ ./bin/mkproject  -x deploy/examples/example-3.xml -s sink -o cr_sink.yaml

Before you deploy these models, edit the autoscaling model, which is specified in the YAML file named cr_source.yaml.
Explicitly limit the CPU resources allocated to speed up the autoscaling. Set the maximum number of replicas to 10, which is the number of partitions that the Kafka topic was configured with.

```yaml
    resources:
      requests:
        memory: "2Gi"
        cpu: "1"
      limits:
        memory: "4Gi"
        cpu: "2"
    autoscale:
      minReplicas: 1
      maxReplicas: 10
```

**Note**: Set the requests.cpu value to 1.  Set the limits.cpu value to 2.
Set the autoscale.maxReplicas value to 10.

Now start the projects. First, start the source project:

    kubectl apply -f cr_source.yaml

Check the usage of the pods as follows:

    [cli]$ kubectl -n cmdline top pods
    NAME                                             CPU(cores)   MEMORY(bytes)
    ...
    source-5f464ddc46-mxnx2                          24m          34Mi

Here, the source pod uses only 24 milli-cpus because there is no data on Kafka
to read and process.

Now start the second project, which floods the Kafka bus with messages.

    kubectl apply -f cr_sink.yaml

After you wait a short amount of time, check the pods again:

    [cli]$ kubectl -n cmdline top pods
    NAME                                             CPU(cores)   MEMORY(bytes)
    ...
    sink-cd7d97d55-txjqs                             2030m        67Mi
    source-5f464ddc46-mxnx2                          1558m        79Mi

You can see that the source is now using 1.5 CPUs. 

Wait a short time and check the pods again.

    [cli]$ kubectl -n cmdline top pods
    NAME                                             CPU(cores)   MEMORY(bytes)
    ...
    sink-cd7d97d55-txjqs                             2703m        88Mi
    source-5f464ddc46-jnkgl                          1273m        80Mi
    source-5f464ddc46-mxnx2                          1785m        132Mi
    source-5f464ddc46-q5t4l                          1922m        93Mi
    source-5f464ddc46-ws4cp                          1010m        74Mi

The project has scaled up to four copies. It should continue to scale up to the maximum
specified (10). Stop the second project (that is writing data to Kafka)
with the following command:

    kubectl delete -f cr_sink.yaml

After a few moments, you should see the number of source instances drop to 1. 
