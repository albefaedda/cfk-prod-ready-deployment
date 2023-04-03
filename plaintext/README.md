## PLAINTEXT deployment

Before deploying the configuration, let's make sure that our topologySpreadConstraint and pod affinity/anti-affinity for Kafka Brokers and ZooKeeper is in place.

### Zookeeper constraints

```yaml
  podTemplate:
    topologySpreadConstraints:
    - maxSkew: 1
      labelSelector:
        matchLabels:
          app: zookeeper
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: DoNotSchedule
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - kafka
          topologyKey: kubernetes.io/hostname  
```

### Kafka Constraints

```yaml
  podTemplate:
    topologySpreadConstraints:
    - maxSkew: 1
      labelSelector:
        matchLabels:
          app: kafka
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: DoNotSchedule
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - zookeeper
          topologyKey: kubernetes.io/hostname
```

Let's also make sure that for both CRs we setup the storage class

```yaml
  storageClass:
    name: production-storage-class
```

Make sure to define the dependency for ZooKeeper in the Kafka CR

```yaml
  dependencies:
    zookeeper: 
      endpoint: zookeeper.confluent.svc.cluster.local:2181
```

Once you're happy with the configuration for Kafka and ZooKeeper, deploy them by running the following command: 

```sh
kubectl apply -f deployment/zk+brokers.yaml
```

Make sure that the pods successfully switch to a running state by running `kubectl get pods -w`.
Once they're running, you can deploy the other Confluent Platform components. 

Make sure the components have the correct Storage Class configured, where needed: 

```yaml
  storageClass:
    name: production-storage-class
```

Set up the dependencies for each component: 

Connect

```yaml
  dependencies:
    kafka:
      bootstrapEndpoint: kafka.confluent.svc.cluster.local:9071
```

KsqlDB

```yaml
  dependencies:
    kafka:
      bootstrapEndpoint: kafka.confluent.svc.cluster.local:9071
```

Schema Registry

```yaml
  dependencies:
    kafka:
      bootstrapEndpoint: kafka.confluent.svc.cluster.local:9071
```

Control Center

```yaml
  dependencies:
    schemaRegistry:
      url: http://schemaregistry.confluent.svc.cluster.local:8081
    ksqldb:
    - name: ksqldb
      url: http://ksqldb.confluent.svc.cluster.local:8088
    connect:
    - name: connect
      url: http://connect.confluent.svc.cluster.local:8083
```

REST Proxy

```yaml
  dependencies:
    kafka:
      bootstrapEndpoint: kafka.confluent.svc.cluster.local:9071
    schemaRegistry:
      url: http://schemaregistry.confluent.svc.cluster.local:8081

```

Now we can deploy these components by using the usual command: 

```sh
kubectl apply -f deployment/other-cp-components.yaml
```

---

## Testing

Since we haven't specified a service to expose the components outside of our Kubernetes cluster, we are going to test Kafka by "exec in" into a Kafka broker pod and create a topic, producing and consuming some messages:

Exec into the Kafka Broker pod:

```sh
kubectl exec -it broker-0 /bin/bash
```

Now, from the bash terminal of the Kafka container we can run some tests making use of the cli tools: 

List Topics:

```sh
kafka-topics --bootstrap-server localhost:9092 --list
```

Create a topic:

```sh
kafka-topics --bootstrap-server localhost:9092 --create --topic test-topic --replication-factor 3 --partitions 1
```

Produce some messages: 

```sh
kafka-console-producer --broker-list localhost:9092 --topic test-topic
> <write your messages here>
```

Consume some messages: 

```sh
kafka-console-consumer --bootstrap-server localhost:9092 --topic test-topic --from-beginning
<the messages in the topic will be shown here>
```
