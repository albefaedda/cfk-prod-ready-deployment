## Authentication with LDAP

To integrate Confluent Platform to authenticate with LDAP, we need to create a user for each CP component in the LDAP server. 

### Deploy OpenLDAP Server

In our example, here are the users created for the CP components in OpenLDAP: 

- kafka-dev
- restproxy-dev
- schemaregistry-dev
- controlcenter-dev
- ksql-dev
- connect-dev
- replicator-dev
- adminrest-dev

These users are in our OpenLDAP [helm configuration](openldap) which we can now deploy to the Kubernetes cluster by issuing the following command:

```sh
helm upgrade --install -f authn-with-ldap/openldap/ldaps-rbac.yaml ldap authn-with-ldap/openldap --namespace confluent 
```

Validate that OpenLDAP is running:

```sh
kubectl get pods --namespace confluent
```

Log in to the LDAP pod:

```sh
kubectl --namespace confluent exec -it ldap-0 -- bash
```

Here are the custom users of ldap: 

```sh
ls -ltr /container/service/slapd/assets/config/bootstrap/ldif/custom
```

Once we have these users created, and OpenLDAP deployed successfully, we need to create a file containing login details for each component and upload them to Kubernetes as secrets. 

### Authentication to LDAP server

To configure the connection and authentication to LDAP, we need to create a read-only user for it. 
The file for this user will look like the following (the username depends on your LDAP configuration):

```properties
username=cn=mds,dc=myorg,dc=com
password=<mds-user-password>
```

Let's call this file `ldap-creds.txt`.
Now, create a Kubernetes secret with the expected key `ldap.txt` and the file `ldap-creds.txt` created in the previous step.

```
kubectl create secret generic ldap-creds \
  --from-file=ldap.txt=./authn-with-ldap/ldap-creds/ldap-creds.txt \
  --namespace confluent
```

We then need to update the Kafka configuration to include the authentication to ldap server: 

```yml
spec:
  identityProvider:                
    type: ldap                     
    ldap:                         
      address: ldap://ldap.confluent.svc.cluster.local:389
      authentication:              
        type: simple
        simple:
          secretRef: ldap-creds
      configurations:
        groupNameAttribute: cn
        groupObjectClass: posixGroup
        groupMemberAttribute: memberuid
        groupMemberAttributePattern: cn=(.*),ou=.*,ou=.*,ou=me,dc=myorg,dc=com
        groupSearchBase: ou=me,dc=myorg,dc=com
        groupSearchScope: 2
        userNameAttribute: cn
        userMemberOfAttributePattern: cn=(.*),ou=.*,ou=.*,ou=me,dc=myorg,dc=com
        userObjectClass: organizationalRole
        userSearchBase: ou=me,dc=myorg,dc=com
        userSearchScope: 2
```

### Kafka LDAP configuration

The file containing the login details for each CP component, has the following format: 

```properties
username=<user>
password=<password>
```

Here's the login information for Kafka, and calling this file `creds-kafka-ldap-user.txt`, we can now upload it to K8S this way: 
Notice that for the kafka user the expected key for the kubernetes secret creation is `plain-broker.txt`. 

```sh
kubectl create secret generic kafka-ldap-user \
  --from-file=plain-interbroker.txt=./authn-with-ldap/ldap-creds/creds-kafka-ldap-user.txt \
  --namespace confluent
```

We now need to update our Kafka listeners, internal and external, to use authentication with LDAP:

```yml
  listeners:
    internal:
      authentication:
        type: ldap       
        jaasConfig:               
          secretRef: kafka-ldap-user 
      tls:
        enabled: true
    external:
      externalAccess:
        type: loadBalancer
        loadBalancer:
          domain: myorg.com
          brokerPrefix: kafka
          bootstrapPrefix: kafka-bootstrap
      authentication:
        type: ldap
      tls:
        enabled: true
```

### Other CP components: 

Here's an example of user file creation for one of the other Confluent Platform components. 
We'll call this file `creds-c3-ldap-user.txt`: 

```properties
username=<user>
password=<password>
```

Create the secret with the expected key `plain.txt` on Kubernetes using the file that contains the credentials: 

```sh
kubectl create secret generic c3-ldap-user \
  --from-file=plain.txt=./authn-with-ldap/ldap-creds/creds-c3-ldap-user.txt \
  --namespace confluent
```

We then update the Confluent Platform component's configuratin with the authentication to Kafka, in this case Control Center:

```yml
spec:
  dependencies:
    kafka:
      bootstrapEndpoint: kafka.confluent.svc.cluster.local:9071
      authentication:
        type: plain
        jaasConfig:
          secretRef: c3-ldap-user
```

## ZooKeeper digest authentication

### Configure ZooKeeper

We can also enable authentication with ZooKeeper, in this case it is of type digest. 
First of all, we need to create a json file containing the credentials that ZooKeeper accepts for authentication: 
In this case, it is only a kafka user: 

```json
{
    "kafka": "kafka-secret"
}
```

Create the Kubernetes secret using the expected key `digest-users.json` and the file `creds-zookeeper-sasl-digest-users.json` you created in the previous step: 

```sh
kubectl create secret generic zk-digest-users \
  --from-file=digest-users.json=./authn-with-ldap/ldap-creds/creds-zookeeper-sasl-digest-users.json \
  --namespace confluent
```

Configure Zookeeper for server-side digest authentication, updating the CR: 

```yml
kind: Zookeeper
spec:
  authentication:
    type: digest
    jaasConfig:
      secretRef: zk-digest-users
```

### Configure Kafka for client-side authentication to Zookeeper

Create a text file containing the user authentication: 

```properties
username=<user>
password=<password>
```

Create the secret with the expected key `digest.txt` and the file created in the previous step `creds-kafka-digest-user.txt`:

```sh
kubectl create secret generic kafka-digest-user-for-zk \
  --from-file=digest.txt=./authn-with-ldap/ldap-creds/creds-kafka-digest-user.txt \
  --namespace confluent
```

Update kafka CR for client-side digest authentication to zookeeper: 

```yml
kind: Kafka
spec:
  dependencies:
    zookeeper:
      authentication:
        type: digest
        jaasConfig:
          secretRef: kafka-digest-user-for-zk
```

--- 

## Deploy the updated components

Update the CRs for each component, now including the configuration for authentication with LDAP on each CP component.
Add the configuration regarding internal and external communication with authentication information. 
Update dependencies for each component to take into account the authentication. 

ZK and Brokers:
```sh
kubectl apply -f authn-with-ldap/deployment/zk+brokers.yaml
```

Other CP Components: 
```sh
kubectl apply -f authn-with-ldap/deployment/other-cp-components.yaml
```

---

## Testing

We can now test authentication to kafka by creating a topic, producing and consuming some messages using the command line tools:
This time, to be able to connect to kafka, we need to provide the authentication information.

Let's create a file containing information about the authentication, together with TLS certificates:

```properties
security.protocol=SSL
ssl.keystore.type=PEM
ssl.keystore.location=/path/to/kafka/kafka-keychain.pem
ssl.key.password=kafka
ssl.truststore.type=PEM
ssl.truststore.location=/path/to/kafka.pem

sasl.mechanism=PLAIN
security.protocol=SASL_SSL
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="testadmin" password="testadmin";
```

Let'save this file as `ldap-config.properties`.

List Topics:

```sh
kafka-topics --bootstrap-server kafka-bootstrap.myorg.com:9092 --list \
  --command-config clients/configs/ldap-config.properties
```

Create a topic:

```sh
kafka-topics --bootstrap-server kafka-bootstrap.myorg.com:9092 --create \
  --topic test-topic --replication-factor 3 \
  --partitions 1 --command-config clients/configs/ldap-config.properties
```

Produce some messages: 

```sh
kafka-console-producer --bootstrap-server kafka-bootstrap.myorg.com:9092 \
  --topic test-topic --producer.config clients/configs/ldap-config.properties 
 <write your messages here>
```

Consume some messages: 

```sh
kafka-console-consumer --bootstrap-server kafka-bootstrap.myorg.com:9092 \
  --topic test-topic --from-beginning --consumer.config clients/configs/ldap-config.properties
 <the messages in the topic will be shown here>
```


Produce some Avro messages:

```sh
kafka-avro-console-producer --bootstrap-server kafka-bootstrap.myorg.com:9092 \
  --topic test-avro-topic \
  --producer.config clients/configs/ldap-config.properties \
  --property schema.registry.url=https://schemaregistry.myorg.com \
  --property value.schema="$(< clients/schema/example-schema.json)" \
  --property schema.registry.ssl.truststore.location=ssl-encryption/certs/generated/schema-registry/sr.pem \
  --property schema.registry.ssl.truststore.type=PEM \
  --property basic.auth.user.info='sr-user:sr-pwd' \
  --property basic.auth.credentials.source="USER_INFO"
 <write your messages here>
```


Consume some Avro messages:

```sh
kafka-avro-console-consumer --bootstrap-server kafka-bootstrap.myorg.com:9092 \
  --topic test-avro-topic --from-beginning \
  --consumer.config clients/configs/ldap-config.properties \
  --property schema.registry.url=https://schemaregistry.myorg.com \
  --property value.schema="$(< clients/schema/example-schema.json)" \
  --property schema.registry.ssl.truststore.type=PEM \
  --property schema.registry.ssl.truststore.location=ssl-encryption/certs/generated/schema-registry/sr.pem \
  --property basic.auth.user.info='sr-user:sr-pwd' \
  --property basic.auth.credentials.source="USER_INFO"
```