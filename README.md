# confluent-schema-registry-tutorial

A tutorial on how to use Confluent Schema registry, from local infrastructure to Confluent Cloud.

A key conceptual aspect I'll address first is how the Schema Subject, Schema ID and Schema Version works. Why? Because the existing documentation is unclear, to me at least. 

Some practical aspects I'll focus on:

1. Starting up the Docker images we need for the exercises - and nothing else
2. Getting to grips with  

## Schema Subject, Schema ID and Schema Version

When you are getting started, chances are high, in my opinion, that you'll do the following and may get confused, like I did:

- Upload a schema for a given subject
- Notice **Schema ID 1** and **Schema Version 1** returned
- Add a field to the schema for the same subject
- Notice **Schema ID 2** and **Schema Version 2** returned

The question becomes, wait, why have a Schema ID AND Schema Version when they are identical? It makes no sense.

### The Schema ID is a Global Technical Construct, Schema Version is Subject Specific

The answer is that, in my opinion, the naming in the Schema Registry is suboptimal. This is what should be returned for clarity:

- Upload a schema for a given subject
- Notice **Global Schema Hashtable Value (GSHV) 1** and **Subject Schema Version 1** returned
- Add a field to the schema for the same subject
- Notice **Global Schema Hashtable Value (GSHV) 2** and **Subject Schema Version 2** returned

Basically, Confluent Schema Registry conceptually does something technical which is to take the MD5 hash of the textual representation 
of the schema to come of with a **Global** hash which serves as a hash table key. If it is found then an attribute of the entry is the
**Global Schema Hashtable Value X**, if not then an entry gets inserted into the hashtable.

So, always remember that the **Schema ID** is shared (aka Global) and it is the true identifier for a schema. The **Schema Version** is in effect used to communicate change outside the scope of the registry, it is a **potential** call to action for both the producer and consumer for a given subject.

I say **potential** call to action because the compatibility setting of the subject determines whether the producer, consumer or both need to change when a schema change gets rolled out.

### References

See [this explanation on StackOverflow](https://stackoverflow.com/a/62010955/433900).

## cURL

In this tutorial we use the Confluent REST Proxy which provides a RESTful interface to an Apache Kafka® cluster.

https://docs.confluent.io/platform/current/kafka-rest/index.html

We use REST Proxy API v3.

https://docs.confluent.io/platform/current/kafka-rest/api.html#crest-api-v3

We also use the Confluent Schema Registry API.

https://docs.confluent.io/platform/current/schema-registry/develop/api.html

### Prove It Locally With cURL

You might not want to sign up for Confluent Cloud just to experiment with the Schema Registry. 

In this section I provide the commands that you and I can use to satisfy ourselves that the Schema ID is a global technical construct 
and that the schema version is subject specific.

TODO finish me

### Other Steps

Get the current mode for Schema Registry at global level.

```
% curl --silent http://localhost:8081/mode | jq
{
  "mode": "READWRITE"
}
```

Get the global compatibility level at global level.

```
% curl --silent http://localhost:8081/config | jq
{
  "compatibilityLevel": "BACKWARD"
}
```

Get the Kafka cluster id:

```
KAFKA_CLUSTER_ID=$(curl -X GET "http://localhost:8082/v3/clusters/" | jq -r ".data[0].cluster_id")
echo $KAFKA_CLUSTER_ID
```

Create transactions topic using the AdminClient functionality of the REST Proxy API v3.:

```
% curl -X POST \
     -H "Content-Type: application/json" \
     -d "{\"topic_name\":\"transactions\",\"partitions_count\":1,\"configs\":[]}" \
     "http://localhost:8082/v3/clusters/${KAFKA_CLUSTER_ID}/topics" | jq .
```

Check whether you can create a topic with key and value schema validation:

```
% curl -X POST -H "Content-Type: application/json" \             
     --data '{
               "topic_name": "transactions",
               "partitions_count": 1,
               "replication_factor": 1,
               "configs": [
                 {"name": "cleanup.policy", "value": "compact"},
                 {"name": "confluent.value.schema.validation", "value": "true"},
                 {"name": "confluent.key.schema.validation", "value": "true"}
               ]
             }' \
     http://localhost:8082/v3/clusters/${KAFKA_CLUSTER_ID}/topics | jq .
```

The above command will only succeed if you are using the Confluent Server image (confluentinc/cp-server), not Apache Kafka (confluentinc/cp-kafka). 
With the latter you will get: 

```
{
  "error_code": 40002,
  "message": "Unknown topic config name: confluent.value.schema.validation"
}
```

Then unless you have specified the Schema Reigstry URL as a cp-server image parameter you will get the following:

```
{
  "error_code": 40002,
  "message": "confluent.key.schema.validation and / or confluent.value.schema.validation is enabled but there is no confluent.schema.registry.url specified at the broker side, will not add the corresponding validator"
}
```

To fix the above, modify docker-compose.yml as follows:

```
broker:
    image: confluentinc/cp-server:7.5.0
    ..
    environment:
      ...
      KAFKA_CONFLUENT_SCHEMA_REGISTRY_URL: 'localhost:8081'
```

The topic creation will now succeed as follows:

```
% curl -X POST -H "Content-Type: application/json" \
     --data '{
               "topic_name": "transactions",   
               "partitions_count": 1,
               "replication_factor": 1,
               "configs": [
                 {"name": "cleanup.policy", "value": "compact"},
                 {"name": "confluent.value.schema.validation", "value": "true"},
                 {"name": "confluent.key.schema.validation", "value": "true"}
               ]
             }' \
     http://localhost:8082/v3/clusters/${KAFKA_CLUSTER_ID}/topics | jq .


{
  "kind": "KafkaTopic",
  "metadata": {
    "self": "http://rest-proxy:8082/v3/clusters/MkU3OEVBNTcwNTJENDM2Qg/topics/transactions",
    "resource_name": "crn:///kafka=MkU3OEVBNTcwNTJENDM2Qg/topic=transactions"
  },
  "cluster_id": "MkU3OEVBNTcwNTJENDM2Qg",
  "topic_name": "transactions",
  "is_internal": false,
  "replication_factor": 1,
  "partitions_count": 1,
  "partitions": {
    "related": "http://rest-proxy:8082/v3/clusters/MkU3OEVBNTcwNTJENDM2Qg/topics/transactions/partitions"
  },
  "configs": {
    "related": "http://rest-proxy:8082/v3/clusters/MkU3OEVBNTcwNTJENDM2Qg/topics/transactions/configs"
  },
  "partition_reassignments": {
    "related": "http://rest-proxy:8082/v3/clusters/MkU3OEVBNTcwNTJENDM2Qg/topics/transactions/partitions/-/reassignment"
  },
  "authorized_operations": []
}
```

Create the initial schema:

```
% curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",\"name\":\"Payment\",\"namespace\":\"io.confluent.examples.clients.basicavro\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"}]}"}' http://localhost:8081/subjects/transactions-value/versions

{"id":1}%                                                
  ```

Now we try and modify the schema by adding a new "region" field. This fails since the new region field does not have a default value and so this is not BACKWARD compatible:

```
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",\"name\":\"Payment\",\"namespace\":\"io.confluent.examples.clients.basicavro\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"},{\"name\":\"region\",\"type\":\"string\"}]}"}' \
  http://localhost:8081/subjects/transactions-value/versions
```

This succeeds since it is BACKWARD compatible given that the new field region has a default value:

```
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",\"name\":\"Payment\",\"namespace\":\"io.confluent.examples.clients.basicavro\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"},{\"name\":\"region\",\"type\":\"string\",\"default\": \"\"}]}"}' \ 
  http://localhost:8081/subjects/transactions-value/versions
```

