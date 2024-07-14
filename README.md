# confluent-schema-registry-tutorial

A tutorial on how to use Confluent Schema registry, from local infrastructure to Confluent Cloud.

A key conceptual aspect I'll address first is how the Schema Subject, Schema ID and Schema Version works. Why? Because the existing documentation is unclear, to me at least. 

Some practical aspects I'll focus on:

1. Starting up the Docker images we need for the exercises - and nothing else.
2. Getting to grips with the cURL commands we'll need to experiment with Schema Subject, Schema ID and Schema Version aspects. So we understand how they work in no uncertain terms.

## Schema Subject, Schema ID and Schema Version

When you are getting started, chances are high, in my opinion, that you'll do the following and may get confused, like I did:

- Upload a schema for a given subject
- Notice **Schema ID 1** and **Schema Version 1** returned
- Add a field to the schema for the same subject
- Notice **Schema ID 2** and **Schema Version 2** returned

The question becomes, wait, why have a Schema ID AND Schema Version when they are identical? It makes no sense.

### The Schema ID is a Global Technical Construct, Schema Version is Subject Specific

The answer is that, in my opinion, the naming in the Schema Registry is suboptimal. This is what should be returned for clarity, though the default brief versions are probably easier once you understand it:

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

### Prove It Locally With cURL and Docker Compose

You might not want to sign up for Confluent Cloud just to experiment with the Schema Registry.

In this section I provide the commands that you and I can use to satisfy ourselves that the Schema ID is a global technical construct 
and that the schema version is subject specific.

The key API references are:

- [Schema Registry API Reference](https://docs.confluent.io/platform/current/schema-registry/develop/api.html)

#### Inspect cp-all-in-one-kraft/docker-compose.yml and docker-compose up -d

All I wanted here was to have the bare minimum number of containers running up to start these images:

- confluentinc/cp-server
- confluentinc/cp-schema-registry
- confluentinc/cp-kafka-rest

I modified an existing Confluent docker-compose.yml file and used the Docker Compose profiles feature to in essence disable the extra containers
by assigning them profiles: [all] (so if you don't use that profile only 3 containers start up).

Sadly there are persistent errors in the logs, I don't have the time to debug them at this time, the net result is that the cp-server image
dies after a few minutes, which is clearly supoptimal - but I still don't want to give up on a **local + cloud** workflow.

#### Get the Kafka cluster id.

```
KAFKA_CLUSTER_ID=$(curl -X GET "http://localhost:8082/v3/clusters/" | jq -r ".data[0].cluster_id")
echo $KAFKA_CLUSTER_ID
```

#### Create the transactions topic with key and value schema validation:

```
% curl -X POST -H "Content-Type: application/json" http://localhost:8082/v3/clusters/${KAFKA_CLUSTER_ID}/topics \
--data-binary @- << EOF    
{
  "topic_name": "transactions",
  "partitions_count": 1,
  "replication_factor": 1,
  "configs": [
    {"name": "cleanup.policy", "value": "compact"},
    {"name": "confluent.value.schema.validation", "value": "true"},
    {"name": "confluent.key.schema.validation", "value": "true"}
  ]
}
EOF
```

#### Create the initial schema for the transactions-value subject.

```
% curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",\"name\":\"Payment\",\"namespace\":\"io.confluent.examples.clients.basicavro\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"}]}"}' http://localhost:8081/subjects/transactions-value/versions

{"id":1}                                                
  ```

#### Retrieves only the schema identified by the input ID.

```
% curl --silent http://localhost:8081/schemas/ids/1/schema | jq
{
  "type": "record",
  "name": "Payment",
  "namespace": "io.confluent.examples.clients.basicavro",
  "fields": [
    {
      "name": "id",
      "type": "string"
    },
    {
      "name": "amount",
      "type": "double"
    }
  ]
}
```

#### Get the subject-version pairs identified by the input ID.

```
% curl --silent http://localhost:8081/schemas/ids/1/versions | jq
[
  {
    "subject": "transactions-value",
    "version": 1
  }
]
```

#### Now create the initial schema for the super-transactions-value subject. 

At this stage we don't have a super-transactions topic. Creating the super-transactions topic is not necessary in terms of our illustration, so I won't do so 
and only create the super-transactions-value subject schema  in the registry.

The **super-transactions-value** subject schema is intentionally identical to the transactions-value subject schema.

```
% curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",\"name\":\"Payment\",\"namespace\":\"io.confluent.examples.clients.basicavro\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"}]}"}' http://localhost:8081/subjects/super-transactions-value/versions
{"id":1}         
```

At this stage you might be wondering why the **id 1** is returned when the **id** is said to be *the globally unique identifier of the schema*.

The reason why is because the Confluent documentation should clarify what *global uniqueness* means, it means uniqueness in terms of the MD5 hash of 
the schema text and NOT the identifier itself.

So, to take another perspective, this call shows that two subjects share the same schema ID, which, given our new understanding, makes sense.

#### Get the subject-version pairs identified by the input ID.

This is the culmination of my illustration in terms of the relationship between the Schema ID and Schema Version. 

You can clearly see that two subjects share the same Schema ID, which makes sense since they are pointing to an identical Schema, we know this is 
the case given our two POST commands illustrated before.

```
% curl --silent http://localhost:8081/schemas/ids/1/versions | jq         
[
  {
    "subject": "transactions-value",
    "version": 1
  },
  {
    "subject": "super-transactions-value",
    "version": 1
  }
]
```

#### Register a new schema under the transactions-value subject.

Essentially, create a new schema.

This succeeds since it is BACKWARD compatible given that the new field region has a default value:

```
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",\"name\":\"Payment\",\"namespace\":\"io.confluent.examples.clients.basicavro\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"},{\"name\":\"region\",\"type\":\"string\",\"default\": \"\"}]}"}' http://localhost:8081/subjects/transactions-value/versions
{"id":2}
```

So, with this, the Subject **transactions-value** has a new Schema ID 2. Let's sanity check that:

```
% curl --silent http://localhost:8081/schemas/ids/2/versions | jq
[
  {
    "subject": "transactions-value",
    "version": 2
  }
]
```

What schema is the **latest** for the Subject **transactions-value**? What is the change history for the **transactions-value** schema?

Use [the call to get a specific version of the schema registered under this subject](https://docs.confluent.io/platform/current/schema-registry/develop/api.html#get--subjects-(string-%20subject)-versions-(versionId-%20version)) but specify **latest** for the **version**.

```
% curl --silent http://localhost:8081/subjects/transactions-value/versions/latest | jq
{
  "subject": "transactions-value",
  "version": 2,
  "id": 2,
  "schema": "{\"type\":\"record\",\"name\":\"Payment\",\"namespace\":\"io.confluent.examples.clients.basicavro\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"},{\"name\":\"region\",\"type\":\"string\",\"default\":\"\"}]}"
}
```

As an aside, notice that there is no schemaType field in the response, as per the documentation this is because the default is AVRO.

Now onto answering the second question. Namely, what is the change history for the **transactions-value** schema?

As you'd expect, we'll use the [Subjects resource](https://docs.confluent.io/platform/current/schema-registry/develop/api.html#subjects). Use the method to [get a list of versions registered under the specified subject]https://docs.confluent.io/platform/current/schema-registry/develop/api.html#get--subjects-(string-%20subject)-versions).

```
% curl --silent http://localhost:8081/subjects/transactions-value/versions            
[1,2]
```

## Broader Schema Registry Experimentation With cURL

In this tutorial we use the Confluent REST Proxy which provides a RESTful interface to an Apache Kafka® cluster.

https://docs.confluent.io/platform/current/kafka-rest/index.html

We use REST Proxy API v3.

https://docs.confluent.io/platform/current/kafka-rest/api.html#crest-api-v3

We also use the Confluent Schema Registry API.

https://docs.confluent.io/platform/current/schema-registry/develop/api.html

### Background Steps

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

Delete topic (because we'll keep on using the same topic name):

```
% curl -X DELETE http://localhost:8082/v3/clusters/${KAFKA_CLUSTER_ID}/topics/transactions
```

Check whether you can create a topic with key and value schema validation:

```
% curl -X POST -H "Content-Type: application/json" http://localhost:8082/v3/clusters/${KAFKA_CLUSTER_ID}/topics \
--data-binary @- << EOF    
{
  "topic_name": "transactions",
  "partitions_count": 1,
  "replication_factor": 1,
  "configs": [
    {"name": "cleanup.policy", "value": "compact"},
    {"name": "confluent.value.schema.validation", "value": "true"},
    {"name": "confluent.key.schema.validation", "value": "true"}
  ]
}
EOF
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

