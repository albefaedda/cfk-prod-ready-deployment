# Confluent Replicator deployment 

In this section we'll see how to configure and deploy Confluent Replicator for CFK to CC cluster replication. 

## Create namespace

First, we create a new namespace and call it `destination` as our Replicator component will be 'Backed to Destination' which is a Confluent Cloud cluster

```sh
kubectl create namespace destination
```

### Switch context to this namespace

```sh
kubectl config set-context --current --namespace destination
```

## Confluent for Kubernetes Operator

We now need to install a new CFK operator in this namespace that will manage the Connect cluster and the Replicator

Deploy CFK Operator: 

```sh
helm upgrade --install confluent-operator \
  /cfk-operator/helm/confluent-for-kubernetes \
  --namespace destination
```

## Secrets for SASL authentication to the Confluent Cloud Kafka cluster and Schema Registry

After having created the destination cluster in Confluent Cloud, enabled Schema Registry and generated corresponding keys, we can create the secrets with these keys in our Kubernetes `destination` namespace.

Destination Kafka cluster SASL API-Key:

```sh
kubectl create secret generic dest-cloud-kafka-sasl \
  --from-file=plain.txt=replicator/confluent-cloud-creds/cc-kafka-creds.txt \
  --namespace destination
  ```

Destination Schema Registry SASL API-Key: 
```sh
  kubectl create secret generic dest-cloud-sr-access \
  --from-file=basic.txt=replicator/confluent-cloud-creds/cc-sr-creds.txt \
  --namespace destination
```

Destination Control Center User:
```sh
  kubectl create secret generic control-center-user \
  --from-file=basic.txt=replicator/confluent-cloud-creds/cfk-c3-creds.txt \
  --namespace destination
```

## Secrets for source cluster TLS certs

Before deploying Replicator, we need to deploy secrets to the `destination` namespace containing the TLS certificates of the source Kafka Cluster and Schema Registry for us to be able to establish connection.
This is the same thing we did in the ssl-encryption section. 

Create Kafka TLS secret: 

```sh
kubectl create secret generic src-kafka-tls-cert \
  --from-file=fullchain.pem=ssl-encryption/certs/generated/kafka/kafka.pem \
  --from-file=cacerts.pem=ssl-encryption/certs/generated/ca/ca.pem \
  --from-file=privkey.pem=ssl-encryption/certs/generated/kafka/kafka-key.pem \
  --namespace destination
```

Create Schema Registry TLS secret:

```sh
kubectl create secret generic src-sr-tls-cert \
  --from-file=fullchain.pem=ssl-encryption/certs/generated/schema-registry/sr.pem \
  --from-file=cacerts.pem=ssl-encryption/certs/generated/ca/ca.pem \
  --from-file=privkey.pem=ssl-encryption/certs/generated/schema-registry/sr-key.pem \
  --namespace destination
  ```

We'll then need to reference these in our replicator configuration

## Connect cluster for Replicator deployment

Now, after having updated the secret reference in our Connect cluster CRD, we can deploy it to Kubernetes

```sh
kubectl apply -f replicator/deployment/src-tls/connect-cluster-for-replicator.yml
```

## Confluent Control Center 

We can also deploy a Control Center component to monitor Replicator

```sh
kubectl apply -f replicator/deployment/src-tls/control-center.yml
```

We can expose Control Center to access it from the browser:

```sh
kubectl --namespace destination port-forward controlcenter-0 9021
```

## Replicator 

We can now deploy our replicator connector for it to consume messages from the source CFK cluster and replicate these into the destination Cloud cluster

```sh
kubectl apply -f replicator/deployment/src-tls/replicator.yml
```