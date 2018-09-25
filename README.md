# kafka-cluster with minikube

Suggested minikube start configuration:
```
minikube start --memory=6144 --cpus=4
```

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

Uses kafka-monitor 1.1.x:
https://hub.docker.com/r/d1egoaz/docker-kafka-monitor/


### Monitor end-to-end a single cluster + JMX

Connect to your local running image:
```bash
kubectl -n kafka-ca1 exec kafka-monitor-{hash} -i -t bash
cd kafka-monitor-1.1.0
```

Run the following command enabling JMX, topic `testtopic` and broker list of the first cluster `kakca-ca1`:
```bash
KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.local.only=false -Dcom.sun.management.jmxremote.port=9999 -Dcom.sun.management.jmxremote.rmi.port=9999 -Djava.rmi.server.hostname=127.0.0.1 " \
  ./bin/end-to-end-test.sh --topic testtopic  --broker-list kafka-0.kafka.kafka-ca1.svc.cluster.local:9092,kafka-1.kafka.kafka-ca1.svc.cluster.local:9092,kafka-2.kafka.kafka-ca1.svc.cluster.local:9092 --zookeeper zookeeper-service.kafka-ca1.svc.cluster.local:2181
```


### Monitor end-to-end multi-cluster + JMX

Connect to your local running image:
```bash
kubectl -n kafka-ca1 exec kafka-monitor-{hash} -i -t bash
cd kafka-monitor-1.1.0
```

Modify `config/multi-cluster-monitor.properties` with the following content:

```json
{
  "multi-cluster-monitor": {
    "class.name": "com.linkedin.kmf.apps.MultiClusterMonitor",
    "topic": "kafka-monitor-topic",

    "produce.service.props": {
      "zookeeper.connect": "zookeeper-service.kafka-ca1.svc.cluster.local:2181",
      "bootstrap.servers": "kafka-0.kafka.kafka-ca1.svc.cluster.local:9092,kafka-1.kafka.kafka-ca1.svc.cluster.local:9092,kafka-2.kafka.kafka-ca1.svc.cluster.local:9092",
      "produce.record.delay.ms": 100,
      "produce.producer.props": {
        "client.id": "kafka-monitor-client-id"
      }
    },

    "consume.service.props": {
      "zookeeper.connect": "zookeeper-service.kafka-ca2.svc.cluster.local:2181",
      "bootstrap.servers": "kafka-0.kafka.kafka-ca2.svc.cluster.local:9092,kafka-1.kafka.kafka-ca2.svc.cluster.local:9092,kafka-2.kafka.kafka-ca2.svc.cluster.local:9092",
      "consume.latency.sla.ms": "20000",
      "consume.consumer.props": {
        "group.id": "kafka-monitor-group-id"
      }
    },

    "topic.management.props.per.cluster" : {
      "first-cluster" : {
        "zookeeper.connect": "zookeeper-service.kafka-ca1.svc.cluster.local:2181",
        "topic-management.topicCreationEnabled": true,
        "topic-management.replicationFactor" : 1,
        "topic-management.partitionsToBrokersRatio" : 2.0,
        "topic-management.rebalance.interval.ms" : 600000,
        "topic-management.topicFactory.props": {
        }
      },

      "last-cluster" : {
        "zookeeper.connect": "zookeeper-service.kafka-ca2.svc.cluster.local:2181",
        "topic-management.topicCreationEnabled": true,
        "topic-management.replicationFactor" : 1,
        "topic-management.partitionsToBrokersRatio" : 2.0,
        "topic-management.rebalance.interval.ms" : 600000,
        "topic-management.topicFactory.props": {
        }
      }
    }

  },

  "reporter-service": {
    "class.name": "com.linkedin.kmf.services.DefaultMetricsReporterService",
    "report.interval.sec": 1,
    "report.metrics.list": [
      "kmf.services:type=produce-service,name=*:produce-availability-avg",
      "kmf.services:type=consume-service,name=*:consume-availability-avg",
      "kmf.services:type=produce-service,name=*:records-produced-total",
      "kmf.services:type=consume-service,name=*:records-consumed-total",
      "kmf.services:type=consume-service,name=*:records-lost-total",
      "kmf.services:type=consume-service,name=*:records-duplicated-total",
      "kmf.services:type=consume-service,name=*:records-delay-ms-avg",
      "kmf.services:type=produce-service,name=*:records-produced-rate",
      "kmf.services:type=produce-service,name=*:produce-error-rate",
      "kmf.services:type=consume-service,name=*:consume-error-rate"
    ]
  },

  "jetty-service": {
    "class.name": "com.linkedin.kmf.services.JettyService",
    "jetty.port": 8000
  },

  "jolokia-service": {
    "class.name": "com.linkedin.kmf.services.JolokiaService"
  }
}
```

Run the following command:
```bash
./bin/kafka-monitor-start.sh config/multi-cluster-monitor.properties
```

## Connect to JMX metrics

Forward the port:
```bash
kubectl -n kafka-ca1 port-forward kafka-monitor-{hash} 9999
```

Connect using jconsole/jvisualvm:
```bash
jconsole localhost:9999
```

## Screenshots

### Cluster view

<img src="resources/namespace-overview.png" width="40%">


### Kafka manager view

<img src="resources/kafka-manaker-kafka-ca1.png" width="40%">


# Changelog

- 2018-09-25, Added kafka-monitor recent image for kafka-monitor 1.1.x
- 2018-09-24, Big refactor, changed per broker config to a single StatefulSet
  - Changed zookeeper image to use the one provided by dockerhub
