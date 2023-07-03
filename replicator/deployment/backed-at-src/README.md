Exec into one of the Kafka brokers

Download CP to use the CLI 
curl -O https://packages.confluent.io/archive/7.3/confluent-community-7.3.0.tar.gz

Unarchive the package
tar xzf confluent-community-7.3.0.tar.gz

```sh
kafka-topics --bootstrap-server localhost:9092 --list
```

Create a topic:

```sh
kafka-topics --bootstrap-server localhost:9092 --create --topic avro-orders --replication-factor 3 --partitions 1
```

Create the file order-schema.json

cat <<EOF >> order-schema.json
{
    "type": "record",
    "namespace": "io.confluent.tutorial",
    "name": "OrderDetail",
    "fields": [
        {"name": "number", "type": "long", "doc": "The order number."},
        {"name": "shipping_address", "type": "string", "doc": "The shipping address."},
        {"name": "subtotal", "type": "double", "doc": "The amount without shipping cost and tax."},
        {"name": "shipping_cost", "type": "double", "doc": "The shipping cost."},
        {"name": "tax", "type": "double", "doc": "The applicable tax."},
        {"name": "grand_total", "type": "double", "doc": "The order grand total ."}
    ]
}
EOF


./kafka-avro-console-producer --bootstrap-server localhost:9092 \
  --topic avro-orders \
  --property schema.registry.url=http://schemaregistry.confluent.svc.cluster.local:8081 \
  --property value.schema="$(< order-schema.json)" 

 ./kafka-avro-console-consumer --bootstrap-server localhost:9092 \
  --topic avro-orders --from-beginning \
  --property schema.registry.url=http://schemaregistry.confluent.svc.cluster.local:8081 \
  --property value.schema="$(< order-schema.json)"


Before running the schema replication, we need to set the Schema registry to a different state: 
SRC to READONLY mode
DEST to IMPORT mode

echo -e "\nSetting srcSchemaregistry to READONLY mode:"
curl -X PUT -H "Content-Type: application/json" --data '{"mode": "READONLY"}' http://schemaregistry.confluent.svc.cluster.local:8081/mode

echo -e "\nSetting destSchemaregistry to IMPORT mode:"
curl -X PUT -H "Content-Type: application/json" --data '{"mode": "IMPORT"}' -u 'D2TGXAOSF2WBC5DE:fEga5YhPfuzpr7tZExNtijKvG0pzuY1m2Vo/bp2RmH6AR7krg7wdnHDMRX1uI5Oj' https://psrc-y5q2k.europe-west3.gcp.confluent.cloud/mode


After completing the Schemas translation, change the SRs state

We shouldn't allow the creation of new schemas in the src SR so we set it to READONLY

echo -e "\nSetting srcSchemaregistry to READONLY mode:"
curl -X PUT -H "Content-Type: application/json" --data '{"mode": "READONLY"}' http://schemaregistry.confluent.svc.cluster.local:8081/mode

echo -e "\nSetting destSchemaregistry to READWRITE mode:"
curl -X PUT -H "Content-Type: application/json" --data '{"mode": "READWRITE"}' -u 'D2TGXAOSF2WBC5DE:fEga5YhPfuzpr7tZExNtijKvG0pzuY1m2Vo/bp2RmH6AR7krg7wdnHDMRX1uI5Oj' https://psrc-y5q2k.europe-west3.gcp.confluent.cloud/mode
