# kafka-cluster with minikube

## Instructions to build the cluster

Run the commands in the following order:

``` bash
kubectl apply -f 00-namespace/
kubectl apply -f 01-zookeeper/
kubectl apply -f 02-kafka/
kubectl apply -f 03-yahoo-kafka-manager/
```

### Configure kafka-manager

Open kafka-manager:

``` bash
open $(minikube service -n kafka-ca1 kafka-manager --url)
```

Add new cluster, and use the following data for `Cluster Zookeeper Hosts`:

```
zookeeper-service:2181
```


## Screenshots

### Cluster view

<img src="resources/namespace-overview.png" width="40%">


### Kafka manager view

<img src="resources/kafka-manaker-kafka-ca1.png" width="40%">
