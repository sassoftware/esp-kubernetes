## ESP Operator

### Introduction

The ESP operator watches for custom resources that the ESP server can run. When the ESP operator detects a new custom resource, it reads the resource and generates a single Kubernetes pod running an ESP Server.  The ESP server runs the project that is 
embedded within the ESPServer custom resource. Further, the ESP operator configures the Ingress endpoint that exposes the REST service for the ESP server.  The ESP server running in the Kubernetes pod uses this endpoint
to communicate externally.

### Examples

#### Read and Write to the Persistent Volume

You need the following files to run this example: 

* esp-cloud/deploy/examples/input/array_input01.csv.gz
* esp-cloud/deploy/examples/example-1.xml

1. Uncompress the input file, deploy/examples/input/array_input01.csv.gz.

2. Copy the expanded file (array_input01.csv) to the directory `input/` on your persistent volume. You can do this with
the filebrowser tool provided in this repository or with any standard Unix or Kubernetes tools.

3. Use the bin/mkproject script to convert the
standard ESP XML project (deploy/examples/example-1.xml) into a custom resource.  When this resource is
applied in the Kubernetes cluster, it triggers the ESP operator to start an ESP
server pod that runs the specified project. 

The syntax is as follows:

```shell
       [~]$ ./bin/mkproject  -x deploy/examples/example-1.xml -s array -o cr_array.yaml

* The -x argument specifes the ESP XML project files
* The -o argument specifies the file to which to write the custom resource
* The -s `<service name>` argument specifies the service name for the project. 
  the endpoint `<namespace>.<domain root>/SASEventStreamProcessingServer/<service name> is created by the ESP operator.
```

4. Apply the custom resource to your Kubernetes cluster:

       [~]$ kubectl -n <namespace> apply -f cr_array.yaml

5. Verify that the pod is running with the following command:

       [~]$ kubectl -n <namespace> get pods

       NAME                                                              READY   STATUS    RESTARTS   AGE
       array-6c57d8699b-l52x6                                            1/1     Running   0          18h
       espfb-deployment-5cc85c6bfd-9fj6n                                 1/1     Running   0          22h
       postgres-deployment-56c9d65d6c-lk4jj                              1/1     Running   1          22h
       sas-esp-operator-86f48f8899-q6bgv                                 1/1     Running   0          22h
       sas-event-stream-processing-metering-app-69fbbffdc7-dnth6         1/1     Running   0          22h

6. Use the filebrowser to inspect the directory `output/`. You should see
the output CSV file (array_output01.csv) that the project has created.

#### Using a Multi-Partitioned Kafka Topic for Autoscaling

This example shows how to feed a project data through a multi-partitioned Kafka topic.
Kubernetes and the ESP operator scale
the number of project pods according to resource load.

Specifically, you deploy a `strimzi kafka operator` to the cluster and create a three-broker
Kafka cluster with a single topic named *sjkinput*. This topic is spread over ten
Kafka partitions. Detailed configuration instructions are beyond the scope of
this document.  

Use the following helpful hints:

1. Install the Kafka operator following the instructions at the [kafka operator](https://operatorhub.io/operator/strimzi-kafka-operator) web site.

2. Apply the following two YAML files to your namespace to do the following:
        * Create the kafka cluster.
        * Instantiate the topic on the cluster.

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

This example runs two SAS Event Stream Processing models.  
* The first model reads from Kafka and then
computes data values.  The model scales up or down automatically.
* The second model reads a CSV file and then loops and loads significant amounts of
data onto Kafka. This forces the first model to consume CPU and then triggers the
autoscaling of the ESP project.

1. Uncompress the input file deploy/examples/input/kafka_input01.csv.gz.

2. Copy the
expanded file (kafka_input01.csv) to the directory `<namespace>/input` on your
persistent volume.

3. Create deployable custom resources for the two models. 

   a. Create the model that reads from a Kafka topic and autoscales.

       [cli]$ ./bin/mkproject  -x deploy/examples/example-2.xml -s source -o cr_source.yaml

   b. Create the model that reads from a CSV file on the persistent volume and pushes messages to
Kafka:

       [cli]$ ./bin/mkproject  -x deploy/examples/example-3.xml -s sink -o cr_sink.yaml

4. Before you deploy these models, edit the autoscaling model.  This is specified in the YAML file named cr_source.yaml.
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
Set the autoscale.maxReplicas to 10.

5. Start the ESP projects.

   a. Start the source project:

      kubectl apply -f cr_source.yaml

      Check the usage of the pods as follows:

      [cli]$ kubectl -n <namespace> top pods
      NAME                                             CPU(cores)   MEMORY(bytes)
      ...
      source-5f464ddc46-mxnx2                          24m          34Mi

      Here, the source pod uses only 24 milli-cpus, because there is no data on Kafka to read and process.

  b. Start the second project that floods the Kafka bus with messages.

      kubectl apply -f cr_sink.yaml

6. Wait a short amount of time and then check the pods again:

    [cli]$ kubectl -n <namespace> top pods
    NAME                                             CPU(cores)   MEMORY(bytes)
    ...
    sink-cd7d97d55-txjqs                             2030m        67Mi
    source-5f464ddc46-mxnx2                          1558m        79Mi

    You can see that the source is now using 1.5 cpus. 

7. Wait a short time and check the pods again.

    [cli]$ kubectl -n <namespace> top pods
    NAME                                             CPU(cores)   MEMORY(bytes)
    ...
    sink-cd7d97d55-txjqs                             2703m        88Mi
    source-5f464ddc46-jnkgl                          1273m        80Mi
    source-5f464ddc46-mxnx2                          1785m        132Mi
    source-5f464ddc46-q5t4l                          1922m        93Mi
    source-5f464ddc46-ws4cp                          1010m        74Mi

    The project has scaled up to four copies. It will continue to scale up to the maximum specified (10). Stop the second project (that is writing data to Kafka) with the following command:

    kubectl delete -f cr_sink.yaml

After a few moments, you should see the number of source instances drop to 1. 
