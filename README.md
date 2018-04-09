# QCon.ai Workshop

Exercises for the QCon Workshop

## Exercise 0: Getting Ready

If you're reading this, you probably know where to find the repo with the instructions, since this is it! Now that you're here, follow these instructions to get ready for the workshop:

1. Install [Docker Compose](https://docs.docker.com/compose/install/) on your system. There are Mac, Windows, and Linux options available at the link.

2. Clone this repo by typing `git clone https://github.com/confluentinc/qcon-ai-workshop` from a terminal.

2. From the `qcon-ai-workshop` directory (which you just cloned), run `docker-compose pull`. This will kick off no small amount of downloading. Get this primed before Exercise 1 begins later on!

## Exercise 1: Producing and Consuming to Kafka topics

1. Run the workshop application by typing `docker-compose up -d` from the project root. This will start up a Kafka broker, a Zookeeper node, and a KSQL container, and a worker container. The "worker" is a helper container with some data and tools in it.

2. Get into the worker container by typing `docker-compose run worker bash`. This will get you a terminal inside the container.

## Exercise 2: Schemas, Schema Registry and Schema Compatibility
In this exercise we'll design an Avro schema, registry it in the Confluent Schema Registry, produce and consume events using this schema, and then modify the schema in compatible and incompatible ways.

We assume you already have the environment up and running from the first exercise.

1. Think of a stream processing use-case that interests you.

   What kind of data do you have? Which topics will you need? select one of the topics and decide on key and value schema for records in the topic. How did the choice of topics influence the event schema? What trade-offs did you make in designing the data model?

2. Write down the schema definition in JSON format.

   You can see the schema definition rules for Avro [here](https://avro.apache.org/docs/1.8.1/spec.html#schemas). If you are stuck coming up with your own schema, you can find a schema that we created for our movies topic in `movies-raw.avsc`.

3. Now, lets register the schema in the Confluent Schema Registry.

   Instructions can be found in [Schema Registry documentation](https://docs.confluent.io/current/schema-registry/docs/intro.html#quickstart).
   We are registering a schema for values, not keys. And in my case, the records will belong to topic `movies-raw`, so I'll register the schema under the subject `movies-raw-value`.
   It is important to note the details of the Schema Registry API for [registering a schema](https://docs.confluent.io/current/schema-registry/docs/api.html#post--subjects-(string-%20subject)-versions). It says:
   ```  
   Request JSON Object:  
    
   schema – The Avro schema string
   ```
   Which means that we need to pass to the API a JSON record, with one key "schema" and the value is a string containing our schema. We can't pass the schema itself when registering it.
   I used `jq` to wrap our Avro Schema appropriately: `jq -n --slurpfile schema movies-raw.avsc  '$schema | {schema: tostring}'`
   And then passed the output of `jq` to `curl` with a pipe:    
   ```  
   jq -n --slurpfile schema movies-raw.avsc  '$schema | {schema: tostring}' | curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" --data @- http://localhost:8081/subjects/movies-raw-value/versions  
   ```  
   The output should be an ID. Remember the ID you got, so you can use it when producing and consuming events.  

4. Now it is time to produce an event with our schema. We'll use the REST Proxy for that.

   You can see [few examples for using Rest Proxy](https://docs.confluent.io/current/kafka-rest/docs/intro.html#produce-and-consume-avro-messages). Note that you don't have to include the entire schema in every single message, since the schema is registered, you can just include the ID: https://docs.confluent.io/current/kafka-rest/docs/api.html#post--topics-(string-topic_name)  
  
   For example, to produce to the movies topic, we can run:  
   ```
   curl -X POST -H "Content-Type: application/vnd.kafka.avro.v2+json" -H "Accept: application/vnd.kafka.v2+json" --data '{"value_schema_id": 1, "records": [{"value": {"movie":{"movie_id": 1, "title": "Ready Player One", "release_year":2018}}}]}'  http://localhost:8082/topics/movies-raw
   ```
  
5. Lets try to consume some messages. 

   I'll use the simple consumer (i.e. read a specific set of messages from a specific partition, rather than subscribe to all new messages in a topic) because it is simple. You can see examples for using the new consumer API in the documentation linked above.
   ```
   curl -H "Accept: application/vnd.kafka.avro.v1+json" http://localhost:8082/topics/movies-raw/partitions/0/messages?offset=0&count=10
   ```

6. Now is the fun part. 

   Make some changes to the schema - add fields, remove fields or modify types. Is the result compatible? Lets check with schema registry:
   ```
   jq -n --slurpfile schema movies-raw-new.avsc  '$schema | {schema: tostring}' |curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" --data @- http://localhost:8081/compatibility/subjects/movies-raw-value/versions/latest
   ```

7. Use the REST Proxy to produce and consume messages with modified schemas, both compatible and in-compatible. What happens?


## Exercise 3: Your Own Schema Design

## Exercise 4: Enriching Data with KSQL
