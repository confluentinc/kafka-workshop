# QCon.ai Workshop

Exercises for the QCon Workshop

## Exercise 0: Getting Ready

If you're reading this, you probably know where to find the repo with the instructions, since this is it! Now that you're here, follow these instructions to get ready for the workshop:

1. Install [Docker Compose](https://docs.docker.com/compose/install/) on your system. There are Mac, Windows, and Linux options available at the link.

2. Clone this repo by typing `git clone https://github.com/confluentinc/qcon-ai-workshop` from a terminal.

3. From the `qcon-ai-workshop` directory (which you just cloned), run `docker-compose pull`. This will kick off no small amount of downloading. Get this primed before Exercise 1 begins later on!

## Exercise 1: Producing and Consuming to Kafka topics

1. Run the workshop application by typing `docker-compose up -d` from the project root. This will start up a Kafka broker, a Zookeeper node, and a KSQL container, and a worker container. The "worker" is a helper container with some data and tools in it.

2. Get into the worker container by typing `docker-compose run worker bash`. This will get you a terminal inside the container.

3. Verify that you have connectivity to your Kafka cluster by typing `kafkacat -b kafka1:9092 -L`. This will list all cluster metadata, which at this point isn't much.

4. Produce a single record into the `movies-raw` topic from the `streams-demo/data/movies-json.js` file. Hint: use `tail -n 1` to pipe a single record to `kafkacat`, and check out the `-P` and `-t` command line switches at `kafkacat --help`.

5. Once you've produced a record to the topic, open up a new terminal tab or window and consume it using `kafkacat` and the `-C` switch.

6. Go back to the producer terminal tab and send two records to the topic using `tail -n 2`. (It's okay that one of these is a duplicate.)

7. For fun, keep the consumer tab visible and run this shell script in the producer tab:
```bash
cat streams-demo/data/movies-json.js | while read in;
do
echo $in | kafkacat -b kafka1:9092 -P -t movies-raw
sleep 1
done
```

8. Be sure to finish up by dumping all movie data into the `movies-raw` topic with `cat movies-json.js | kafkacat -b kafka1:9092 -P -t movies-raw`.

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

1. Break into groups of two to four people. In your groups, discuss some simple business problems within each person's domain. 

2. Agree on one business problem to model. Draw a simple entity diagram of it, and make a list of operations your application must perform on the model. For example, if your business is retail, you might make a simple model of inventory, with entities for location, item, and supplier, plus operations for receiving, transferring, selling, and analysis.

3. Sketch out a simple application to provide a front end and necessary APIs for the system. For each service in the application, indicate what interface it provides (web front-end, HTTP API, etc.) and what computation it does over the data.

4. Some of the entities in your model are truly static, and some are not. Revisit your entity model and decide which entities are should be streams and which are truly tables.

5. Re-draw your diagram from step three with the appropriate tables replaced by streams. For each service in the application, keep its interface constant, but re-consider what computation it does over the data.

6. Time permitting, present your final architecture to the class. Explain how you adjudicated each stream/table duality and what streaming computations you planned.

## Exercise 4: Enriching Data with KSQL

1. In separate terminal tabs, keep the worker container from exercise #1 running, and run a second container with the following steps:
```
$ docker-compose run ksql bash
root@929aa798b628:/# ksql-cli local --properties-file=ksqlserver.properties
```

2. In the worker container, see the movies data with `head -n 1 streams-demo/data/movies-json.js  | kafkacat -b kafka1:9092 -t movies-raw -P`


3. In the KSQL container, create a stream around the raw movie data: `CREATE STREAM movies_src (movie_id LONG, title VARCHAR, release_year INT) WITH (VALUE_FORMAT='JSON', KAFKA_TOPIC='movies-raw');`

4. As you can see by selecting the records from that stream, the key is null. Re-key it with `CREATE STREAM movies_rekeyed AS SELECT * FROM movies_src PARTITION BY movie_id;`

5. Run a non-persistent select on `movies_rekeyed` in the KSQL window, then stream ten more movie records into the `movies-raw` topic. Watch them appear in the rekeyed stream. 

6. Turn the movies into a table of reference data with `CREATE TABLE movies_ref (movie_id LONG, title VARCHAR, release_year INT) WITH (VALUE_FORMAT='JSON', KAFKA_TOPIC='MOVIES_REKEYED', KEY='movie_id');`

7. In a new worker container (another terminal tab with `docker-compose run worker bash`), do the following:
```
cd streams-demo
./gradlew streamJsonRatings
```

8. Attempt to join ratings to the movie data with `SELECT m.title, m.release_year, r.rating FROM ratings r LEFT OUTER JOIN movies_ref m on r.movie_id = m.movie_id;`. Note the nulls! We need more movies in the reference stream.

8. `cat` the rest of the `movies-json.js` file into the stream. Notice that the join starts working!

9. Create a table containing average ratings as follows: `CREATE TABLE movie_ratings AS SELECT m.title, SUM(r.rating)/COUNT(r.rating) AS avg_rating, COUNT(r.rating) AS num_ratings FROM ratings r LEFT OUTER JOIN movies_ref m ON m.movie_id = r.movie_id GROUP BY m.title;`

10. Select from that table and inspect the average ratings. Do you agree with them? Discuss. (If you want the table to stop updating, kill the Gradle task that is streaming the ratings—it's been going this whole time.)
