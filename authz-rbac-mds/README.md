## Authorisation with RBAC

When we configure RBAC for authorisation, we need another service called MDS (MetaData Service) to manage connection with LDAP. 
To deploy this component, we need to create a (MDS token)[https://docs.confluent.io/platform/current/kafka/configure-mds/index.html#create-a-pem-key-pair] for use by the token service. 

### MDS Token 

Create the 2048-bit RSA private key, and store the keys in a folder named <path-to-tokenKeypair.pem>

```sh
mkdir <path-to-tokenKeypair.pem> && openssl genrsa -out <path-to-tokenKeypair.pem> 2048
```

Extract the public key to a folder <path-to-public.pem>

```sh
openssl rsa -in <path-to-tokenKeypair.pem> -outform PEM -pubout -out <path-to-public.pem>
```

Create a Kubernetes secret with the MDS token just created: 

```sh
kubectl create secret generic mds-token \
  --from-file=mdsPublicKey.pem=./authz-rbac-mds/mds-token/mds-publickey.txt \
  --from-file=mdsTokenKeyPair.pem=./authz-rbac-mds/mds-token/mds-tokenkeypair.txt \
  --namespace confluent
```

--- 

### Enable MDS for authorisation

To enable MDS, we need to update our Kafka component: 

```yml
kind: Kafka
spec:
  authorization:
    type: rbac
    superUsers:
      - "User:kafka"
      - "User:testadmin"
  services:
    mds: 
      tokenKeyPair:
        secretRef: mds-token
      provider:
        type: ldap
        ldap: 
          address: ldap://ldap.confluent.svc.cluster.local:389
          authentication:
            type: simple
            simple:
              secretRef: ldap-creds   # the same secret created earlier as part of the authentication
          configurations:
            groupNameAttribute: cn
            groupObjectClass: posixGroup
            groupMemberAttribute: memberuid
            groupMemberAttributePattern: cn=(.*),ou=.*,ou=.*,ou=ae,dc=test,dc=com
            groupSearchBase: ou=ae,dc=test,dc=com
            groupSearchScope: 2
            userNameAttribute: cn
            userMemberOfAttributePattern: cn=(.*),ou=.*,ou=.*,ou=ae,dc=test,dc=com
            userObjectClass: organizationalRole
            userSearchBase: ou=ae,dc=test,dc=com
            userSearchScope: 2
  dependencies:
    kafkaRest: 
      authentication:
        type: bearer 
        bearer:
          secretRef: kafka-mds-client # created in the next section (Kafka)

```

The `kafka-mds-client` secret here is a bearer authentication to MDS service. We'll see next how to create these.

### Bearer authentication

When MDS is enabled, CFK always uses bearer authentication for all CP components. 
Create a text file with the credentials for MDS for each Confluent Platform component.
Since MDS can be integrated with LDAP, we are going to use the same credentials we have for authentication with LDAP but this time we need to create these as bearer authentication. 

Create a file, for each CP component containing its credential for MDS/LDAP. 

```txt
username=<cp-component-username>
password=<cp-component-password>
```

We now need to create a secret with the expected key `bearer.txt` and the file created in the previous section, for each component:

```sh
kubectl create secret generic kafka-mds-client \
  --from-file=bearer.txt=/authz-rbac-mds/mds-bearer/kafka-mds-client.txt \
  --namespace confluent
```

### Kafka REST Admin

Together with MDS, we need to deploy another component called "REST Admin" which is used for Admin tasks when RBAC is enabled.

Here's the deployment configuration for this component: 

```yml
apiVersion: platform.confluent.io/v1beta1
kind: KafkaRestClass
metadata:
  name: default
  namespace: confluent
spec:
  kafkaRest:
    authentication:
      type: bearer               
      bearer:
        secretRef: rest-credential
```

The `rest-credential` secretRef is a secret containing the LDAP authentication for the Admin REST API.

For the REST Admin, when we create the secret containing the credentials, we need to specify it in the following way: 

```sh
kubectl create secret generic rest-credential \
  --from-file=bearer.txt=/authz-rbac-mds/mds-bearer/admin-rest-mds-client.txt \
  --from-file=basic.txt=/authz-rbac-mds/mds-bearer/admin-rest-mds-client.txt \
  --namespace confluent
```

### MDS dependency

Once we enable MDS, we need to add it to the dependencies of each Confluent Platform component because each of them needs to use MDS to authenticate and get the correct authorisation through RBAC. 
Here's the confiugration snippet to be added to each component: 

```yml
  dependencies:
    mds:
      endpoint: https://kafka.confluent.svc.cluster.local:8090
      tokenKeyPair:
        secretRef: mds-token
      authentication:
        type: bearer               
        bearer:
          secretRef: <component-mds-bearer>
```

## RBAC Roles definition

Once we have our Confluent Platform components deployed with RBAC enabled, we can create some roles for our users/service-accounts. 

A service-account is a special user that exists in LDAP used for applications that read/write data into Kafka topics so usually have RBAC roles that allow them to create/read/write data. In our example, we have a `testadmin` user for which we created a ResourceOwner role that allows it to create/read/write data on topics starting with test. An example of role definition is [here](roles/testadmin-topics-owner.yml). 
Another [example](roles/testadmin-sr-subjects-owner.yml) is for the same service-account to be a ResourceOwner for Schema Registry Subjects that start with `test` so that it can create schemas under that subject. 

A user is a person that exists in LDAP and has permissions to access Confluent Control Center to manage the clusters of certain Confluent Platform components, or manage other users authorisations, for example.
An [example](roles/alice-cp-admin.yml) of a user role binding definition is for our user alice to authenticate to Control Center and be able to view information of Kafka cluster, Schema Registry cluster, KsqlDB cluster and Connect cluster

## Testing

Now that we have our user and service-account authorised to perform some work on the platform, we can run some tests similarly to how we did previously with LDAP. We can actually reuse the same configurations and commands used then.

Below are listed again the configurations and commands used for the testing: 

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
  --command-config configs/ldap-config.properties
```

Create a topic:

```sh
kafka-topics --bootstrap-server kafka-bootstrap.myorg.com:9092 --create \
  --topic test-topic --replication-factor 3 \
  --partitions 1 --command-config configs/ldap-config.properties
```

Produce some messages: 

```sh
kafka-console-producer --bootstrap-server kafka-bootstrap.myorg.com:9092 \
  --topic test-topic --producer.config configs/ldap-config.properties 
 <write your messages here>
```

Consume some messages: 

```sh
kafka-console-consumer --bootstrap-server kafka-bootstrap.myorg.com:9092 \
  --topic test-topic --from-beginning --consumer.config configs/ldap-config.properties
 <the messages in the topic will be shown here>
```


Produce some Avro messages:

```sh
kafka-avro-console-producer --bootstrap-server kafka-bootstrap.myorg.com:9092 \
  --topic <a-topic-name> --producer.config configs/ldap-config.properties
  --property schema.registry.url=https://schemaregistry.myorg.com \
  --property value.schema="$(< clients/schema/example-schema.json)" \
  --property schema.registry.ssl.truststore.location=ssl-encryption/certs/generated/schema-registry/schema-registry.pem \
  --property schema.registry.ssl.truststore.type=PEM \
  --property basic.auth.user.info="sr-user:sr-password" \
  --property basic.auth.credentials.source="USER_INFO"
 <write your messages here>
```


Consume some Avro messages:

```sh
kafka-avro-console-consumer --bootstrap-server <your-kafka-bootstrap-server> \
  --topic <a-topic-name> --from-beginning \
  --property schema.registry.url=https://schemaregistry.myorg.com \
  --property value.schema="$(< clients/schema/example-schema.json)" \
  --property schema.registry.ssl.truststore.type=PEM \
  --property schema.registry.ssl.truststore.location=ssl-encryption/certs/generated/schema-registry/schema-registry.pem \
  --consumer.config configs/ldap-config.properties \
  --property basic.auth.user.info="sr-user:sr-password" \
  --property basic.auth.credentials.source="USER_INFO"
```