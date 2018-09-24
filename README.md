# kafka-cluster with minikube

## Instructions to build the cluster

Run the commands in the following order:

``` bash
kubectl apply -f 00-namespace/
kubectl apply -f 01-zookeeper/
kubectl apply -f 02-kafka/
kubectl apply -f 03-yahoo-kafka-manager/
kubectl apply -f 04-kafka-monitor/

# depends on creating a new cluster: kafka-ca2
# replace in all previous files s/kafka-ca1/kafka-ca2 and run steps: 00, 01, and 02.
kubectl apply -f 05-kafka-mirrormaker/
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

## Monitor cluster resources

``` bash
watch -n 1 kubectl -n kafka-ca1 get deployments
watch -n 1 kubectl -n kafka-ca1 get statefulsets
watch -n 1 kubectl -n kafka-ca1 get services
watch -n 1 kubectl -n kafka-ca1 get pods
watch -n 1 minikube service list --namespace kafka-ca1
```

## Logs
```bash
kubectl -n kafka-ca1 exec kafka-0 -- tail -f /opt/kafka/logs/state-change.log
kubectl -n kafka-ca1 exec kafka-0 -- tail -f /opt/kafka/logs/server.log
kubectl -n kafka-ca1 exec kafka-0 -- tail -f /opt/kafka/logs/controller.log
```

## kafka-monitoring

```
cd kafka-monitor
./bin/end-to-end-test.sh --topic test-topic --broker-list kafka-0.kafka-ca1.svc.cluster.local:9092,kafka-1.kafka-ca1.svc.cluster.local:9092,kafka-2.kafka-ca1.svc.cluster.local:9092 --zookeeper zookeeper-service.kafka-ca1.svc.cluster.local:2181
```

## Screenshots

### Cluster view

<img src="resources/namespace-overview.png" width="40%">


### Kafka manager view

<img src="resources/kafka-manaker-kafka-ca1.png" width="40%">


# Changelog

- 2018-09-24, Big refactor, changed per broker config to a single StatefulSet
  - Changed zookeeper image to use the one provided by dockerhub
