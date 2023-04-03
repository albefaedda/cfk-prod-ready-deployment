## SSL encryption deployment

### Create SSL/TLS Certificates

#### Create Certificate Authority

This command creates a `ca-key.pem`, `ca.csr` and `ca.pem` in path `ssl-encryption/certs/generated/ca`

```sh
mkdir ssl-encryption/certs/generated/ca && cfssl gencert -initca ssl-encryption/certs/config/ca-csr.json | cfssljson -bare ssl-encryption/certs/generated/ca/ca -
```
Validate Certificate Authority using the command `openssl x509 -in security/tls/generated/ca/ca.pem -text -noout`

#### Create Certificates (with appropriate [SANs](https://docs.confluent.io/operator/current/co-network-encryption.html#configure-network-encryption-with-co-long)) for each component

- Example for Zookeeper certificate

This command creates zk.pem, zk-key.pem and zk.csr

```sh
cfssl gencert -ca=ssl-encryption/certs/generated/ca/ca.pem \
-ca-key=ssl-encryption/certs/generated/ca/ca-key.pem \
-config=ssl-encryption/certs/config/ca-config.json \
-profile=server ssl-encryption/certs/config/zookeeper-domain.json | cfssljson -bare ssl-encryption/certs/generated/zookeeper/zk
```

Validate created certificate and SANs using the following command:

```sh
openssl x509 -in ssl-encryption/certs/generated/zookeeper/zk.pem -text -noout
```

Create the Kubernetes secret for the certicicate: 

```sh
kubectl create secret generic zk-tls-cert \
  --from-file=fullchain.pem=ssl-encryption/certs/generated/zookeeper/zk.pem \
  --from-file=cacerts.pem=ssl-encryption/certs/generated/ca/ca.pem \
  --from-file=privkey.pem=ssl-encryption/certs/generated/zookeeper/zk-key.pem \
  --namespace confluent
```

Repeat the same procedure for the other components (Kafka, Schema Registry, Connect, Control Center, KsqlDB, REST Proxy), commands [here](certs/commands.txt)

--- 

### Deploy the updated components

Update the CRs for each component, including the reference to the secret for the TLS certificate.
Add the configuration regarding internal and external communication with domain information. 
Update dependencies for each component to take into account the TSL. 

ZK and Brokers:
```sh
kubectl apply of ssl-encryption/deployment/zk+brokers.yaml
```

Other CP Components: 
```sh
kubectl apply of ssl-encryption/deployment/other-cp-components.yaml
```

---

### Testing

Now that our components are exposed to the outside world in a secured way (TLS), we can run some tests by reaching the components from outside Kubernetes. 

We can verify that Control Center is up and running by calling it from our browser at `https://controlcenter.myorg.com`

We can then create a topic, and produce and consume some messages using the command line tools:

This time, to be able to connect to kafka, we need to provide the certificates.

Let's create a file containing information about the certificates:

First, we need to encrypt the private key so that we can use it for our client call:

Encrypt private key (server-key.pem)
openssl pkcs8 -topk8 -in server-key.pem -out encryptedPrivateKey.p9 
(use a password when requested)

Create a key chain file, containing: 

1. encrypted key (encryptedPrivateKey.p9)
2. server certificate (server.pem)
3. Certificate Authority (ca.pem)

```properties
security.protocol=SSL
ssl.keystore.type=PEM
ssl.keystore.location=/path/to/kafka/kafka-keychain.pem
ssl.key.password=kafka
ssl.truststore.type=PEM
ssl.truststore.location=/path/to/kafka.pem
```

List Topics:

```sh
kafka-topics --bootstrap-server kafka-bootstrap.myorg.com:9092 --list \
  --command-config ssl-encryption/ssl-client.properties
```

Create a topic:

```sh
kafka-topics --bootstrap-server kafka-bootstrap.myorg.com:9092 --create \
  --topic test-topic --replication-factor 3 \
  --partitions 1 --command-config ssl-encryption/ssl-client.properties
```

Produce some messages: 

```sh
kafka-console-producer --bootstrap-server kafka-bootstrap.myorg.com:9092 \
  --topic test-topic --producer.config ssl-encryption/ssl-client.properties
> <write your messages here>
```

Consume some messages: 

```sh
kafka-console-consumer --bootstrap-server kafka-bootstrap.myorg.com:9092 \
  --topic test-topic --from-beginning --consumer.config ssl-encryption/ssl-client.properties
<the messages in the topic will be shown here>
```


Produce some Avro messages:

```sh
kafka-avro-console-producer --bootstrap-server kafka-bootstrap.myorg.com:9092 \
  --topic test-topic --producer.config ssl-encryption/ssl-client.properties
  --property schema.registry.url=https://schemaregistry.myorg.com \
  --property value.schema="$(< clients/schema/example-schema.json)" \
  --property schema.registry.ssl.truststore.location=ssl-encryption/certs/generated/schema-registry/sr.pem \
  --property schema.registry.ssl.truststore.type=PEM
> <write your messages here>
```


Consume some Avro messages:

```sh
kafka-avro-console-consumer --bootstrap-server <your-kafka-bootstrap-server> \
  --topic <a-topic-name> --from-beginning \
  --property schema.registry.url=https://schemaregistry.myorg.com \
  --property value.schema="$(< clients/schema/example-schema.json)" \
  --property schema.registry.ssl.truststore.type=PEM \
  --property schema.registry.ssl.truststore.location=ssl-encryption/certs/generated/schema-registry/sr.pem
  --consumer.config ssl-encryption/ssl-client.properties
```