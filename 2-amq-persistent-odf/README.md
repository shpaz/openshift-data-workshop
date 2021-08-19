# Exercise 2 - Create a resilient Kafka cluster using ODF RBD 

## Table of Contents

- [Objective](#objective)
- [Diagram](#diagram)
- [Guide](#guide)
- [Takeaways](#takeaways)

# Objective

Getting to know better with partition allocation and logDirs persistency:
- Deploy a Kafka cluster using Openshift Data Foundation RBD as block backend for Kafka logDirs
- Understand parition allocation across Kafka nodes, In terms of failure and rebalancing 
- Understand how we can scale our cluster seamlessly in an automated way 
- Understand how operators help us in managing day2 operations 

# Diagram

![](https://miro.medium.com/max/1400/1*_PmX-e1oytvjmIk8mqK8Gw.png)

Make sure you connect to the cluster before starting this exercise! 

# Guide

## Step 1

 Click the `Add+` button in order to consume a resource from Openshift's marketplace. Pick the `Operator Backed` button in order to consume As-A-Service Kafka cluster. 

 Pick the `Kafka` resource and hit `Create` to start the deployment:  

![](../1-explore-amq-operator/pictures/create-kafka.png)

## Step 2 

Hit the `Create` button in order to complete the installation (Make sure to seitch to `persistent-claim` to consume persistent storage for both `Kafka` and `ZooKeeper`): 

![](../1-explore-amq-operator/pictures/kafka-persistent.png)

In Addition, switch to the `YAML View` section and add the following line under the `Storage` section (**For both `kafka` and `zookeeper`**):

```bash
storage:
  type: persistent-claim
  size: 2Gi
```

## Step 3 

Make sure your Kafka cluster was successfully installed and that you can see all of its compnents: 

![](../1-explore-amq-operator/pictures/my-kafka-cluster.png)


## Step 4 

Veirfy that your Kafka cluster installation had been successful by using the `Project -> Pods` in the inventory: 

![](../1-explore-amq-operator/pictures/get-pods.png)

## Step 5

Ensure that the Kafka nodes are indeed using an persistent volumes for storing the Kafka logDirs by using `Project -> PVCs` on the left tab: 

![](../1-explore-amq-operator/pictures/persistent-pvcs.png)


## Step 7 

Having our Kafka cluster using emptyDirs means that on failure that Kafka cluster will have to replicate data on his own because **the underlying volume is gone**.
In this case we will count on Kafka's replication mechanism for replicating the data which can sometimes cause unwanted latency. 

To do so, we'll create a producer a Topic, a Producer and a Consumer that will send messages to one another. Producer --> Topic --> Consumer. 

![](https://miro.medium.com/max/700/1*TX19sKucFmSXSE-k6aMThg.png)

We'll kill one of the Kafka nodes and see how it affects the offset being transfered between the producer and the consumer. 


## Step 8 

Let's create a Kafka topic using the `Add+ -> Operator Backed -> Kafka Topic -> Create` with the name `my-topic`: 

![](../1-explore-amq-operator/pictures/create-topic.png)


Make sure you leave the default values and hit the `Create` button. 

## Step 9 

Validate that the created Kafka topic was created successfuly by using `get kt` command (only if you have the `oc` command-line): 

```bash 
$ oc get kt
                                                                                
NAME       PARTITIONS   REPLICATION FACTOR
my-topic   12           3
```

The Kafka topic was created with 12 parititions and replication factor of 3. 

## Step 10 

Let's create a Kafka user to interact with the created topic, move through the `KafkaUser` CR to verify that you understand how user management is handled in AMQ.

Go to `Add+ -> Operator Backed -> Kafka User` in order to create the `Kafka User`: 
 

![](../1-explore-amq-operator/pictures/create-user.png)

Before you hit the `Create` button, switch to the `YAML View` section to verify you understand all the ACLs that is being given to our created user.

## Step 11

Now let's create a Kafka Producer that will write messages to our `my-topic` topic, and a consumer that will consume those messages via `Add+ -> YAML` 

```bash 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello-world-producer
  name: hello-world-producer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world-producer
  template:
    metadata:
      labels:
        app: hello-world-producer
    spec:
      containers:
      - name: hello-world-producer
        image: strimzici/hello-world-producer:support-training
        env:
          - name: CA_CRT
            valueFrom:
              secretKeyRef:
                name: my-cluster-cluster-ca-cert
                key: ca.crt
          - name: USER_CRT
            valueFrom:
              secretKeyRef:
                name: my-user
                key: user.crt
          - name: USER_KEY
            valueFrom:
              secretKeyRef:
                name: my-user
                key: user.key
          - name: BOOTSTRAP_SERVERS
            value: my-cluster-kafka-bootstrap:9093
          - name: TOPIC
            value: my-topic
          - name: DELAY_MS
            value: "5000"
          - name: LOG_LEVEL
            value: "INFO"
          - name: MESSAGE_COUNT
            value: "5000"
```
Now let's create a Kafka consumer that will read messages from our `my-topic` topic via `Add+ -> YAML` :

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello-world-consumer
  name: hello-world-consumer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world-consumer
  template:
    metadata:
      labels:
        app: hello-world-consumer
    spec:
      containers:
      - name: hello-world-consumer
        image: strimzici/hello-world-consumer:support-training
        env:
          - name: CA_CRT
            valueFrom:
              secretKeyRef:
                name: my-cluster-cluster-ca-cert
                key: ca.crt
          - name: USER_CRT
            valueFrom:
              secretKeyRef:
                name: my-user
                key: user.crt
          - name: USER_KEY
            valueFrom:
              secretKeyRef:
                name: my-user
                key: user.key
          - name: BOOTSTRAP_SERVERS
            value: my-cluster-kafka-bootstrap:9093
          - name: TOPIC
            value: my-topic
          - name: GROUP_ID
            value: my-group
          - name: LOG_LEVEL
            value: "INFO"
          - name: MESSAGE_COUNT
            value: "5000"
```

## Step 13 

Verify that both consumer and producer works as expected by browsing their logs with `Topology -> hello-producer/consumer -> Resources - View Logs`, for example: 

### Producer Logs
![](../1-explore-amq-operator/pictures/producer-logs.png)

### Consumer Logs

![](../1-explore-amq-operator/pictures/consumer-logs.png)


## Step 14 

Now after that we have our producer and consumer running as expected, open another Let's try to delete one of our Kafka pods and see what happens.

From the `Topology -> my-cluster-kafka -> Resources`, Click on one of the pods and hit `Actions -> Delete Pod`: 

![](../1-explore-amq-operator/pictures/delete-pod.png)


Go back to you cosumer logs and verify that you don't see that your cluster recovered much faster, a thing that can take quite a lot of time to rebalance the data. 

### Pause to Think  

*How did this happen? How the broker know which volume to pick? why we didn't see any new volume being created? why wasn't the volume deleted?*


## Step 15 

Now, Let's scale our cluster to see how easy it is to perform day2 operations with our cluster. 

In order to do so, go to `Search -> Resources -> Kafka -> my-cluster -> YAML` and scale the number of `Kafka` replicas to `4` and hit `Save`. 

Make sure that your cluster is starting the rollout process, and that a new `Kafka` node was added to the cluster by going to `Topology -> my-cluster-kafka -> : 

![](../1-explore-amq-operator/pictures/kafka-rollout.png)

*Note: Pay attention to the fact that the entire cluster rolled-out one by one so that we won't have any down time, you can try and login the your consumer's logs once again to verify all data is still there*

Delete the exercise's resources using:
*  `Topology -> hello-producer -> Delete Deployment`
*  `Topology -> hello-consumer -> Delete Deployment`
*  `Search -> Resources -> KafkaUser -> Delete KafkaUser`
*  `Search -> Resources -> KafkaTopic -> Delete all topics`
*  `Search -> Resources -> Kafka -> Delete Kafka`

Make sure you have nothing in the `Topology View`.

*Question*: Were PVs deleted or not? If not, why? (Try to remember what we talked about for `Volume Reclamation Policies`) 

Delete all PVCs: 
*  `Project -> PVC -> Delete`

# Complete

Congratulations! You have completed the second exercise :)

---
[Click Here to return to the Workshop Index](../README.md)
