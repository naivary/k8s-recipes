# Kafka in Kubernetes

The recommended approach to install and use Kafka in Kubernetes is using the
`strimzi` operator. Strimzi can be installed using Helm.

```bash
helm repo add strimzi https://strimzi.io/charts
helm install strimzi strimzi/strimzi-kafka-operator --version 0.51.0
```

To have a simple Kafka cluster you can use the following manifests.

```yaml
kind: Kafka
metadata:
  name: vfde
  namespace: strimzi
spec:
  kafka:
    version: 4.2.0
    metadataVersion: 4.2-IV1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
  entityOperator:
    topicOperator: {}
    userOperator: {}
---
apiVersion: kafka.strimzi.io/v1
kind: KafkaNodePool
metadata:
  name: dual-role
  namespace: strimzi
  labels:
    strimzi.io/cluster: vfde
spec:
  replicas: 1
  roles:
    - controller
    - broker
  storage:
    class: local-path
    type: persistent-claim
    size: 10Gi
```
