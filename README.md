# Kafka Workshop

Exercises for the Kafka Workshop

## Exercise 0: Getting Ready

If you're reading this, you probably know where to find the repo with the instructions, since this is it! Now that you're here, follow these instructions to get ready for the workshop:

1. Install [Docker Compose](https://docs.docker.com/compose/install/) on your system. There are Mac, Windows, and Linux options available at the link.

2. Clone this repo by typing `git clone https://github.com/confluentinc/kafka-workshop` from a terminal.

3. From the `kafka-workshop` directory (which you just cloned), run `docker-compose pull`. This will kick off no small amount of downloading. Get this primed before Exercise 1 begins later on!

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


## Exercise 2: Kafka Connect

The Docker Compose environment includes a Postgres database called `workshop`, pre-populated with a `movies` table. Using Kafka Connect and the JDBC connector you can stream the contents of a database table, along with any future changes, into a Kafka topic. 

First, let's check that Kafka Connect has started up. Run the following:

```bash
$ docker-compose logs -f connect|grep "Kafka Connect started"
```

Wait until you see the output `INFO Kafka Connect started (org.apache.kafka.connect.runtime.Connect)`. Press Ctrl-C twice to cancel and return to the command prompt. 

Now create the JDBC connector, by sending the configuration to the Connnect REST API. *Before running this, make sure that you are in the `kafka-workshop` folder*. 

```bash
curl -i -X POST -H "Accept:application/json" \
        -H  "Content-Type:application/json" http://localhost:8083/connectors/ \
        -d @connect/postgres-source.json
```

If you have [`jq`](https://stedolan.github.io/jq/) on your local machine, you can use the following bash snippet to use the REST API to easily see the status of the connector that you've created. 

```
curl -s "http://localhost:8083/connectors"| jq '.[]'| xargs -I{connector_name} curl -s "http://localhost:8083/connectors/"{connector_name}"/status"| jq -c -M '[.name,.connector.state,.tasks[].state]|join(":|:")'| column -s : -t| sed 's/\"//g'| sort
```

You should get output that looks like this: 

```
jdbc_source_postgres_movies  |  RUNNING  |  RUNNING
```

The JDBC connector will have pulled across all existing rows from the database into a Kafka topic. Run the following, to list the current Kafka topics: 

```bash
docker-compose exec kafka1 bash -c 'kafka-topics --zookeeper zookeeper:2181 --list'
```

You should see, amongst other topics, one called `postgres-movies`. Now let's inspect the data on the topic. Because Kafka Connect is configured to use Avro serialisation we'll use the `kafka-avro-console-consumer` to view it: 

```bash
docker-compose exec connect \
                    kafka-avro-console-consumer \
                    --bootstrap-server kafka1:9092 \
                    --property schema.registry.url=http://schemaregistry:8081 \
                    --topic postgres-movies --from-beginning
```

You should see the contents of the movies table spooled to your terminal. 

**Leave the above command running**, and then in a new window launch a postgres shell: 

```bash
docker-compose exec database bash -c 'psql --username postgres --d WORKSHOP'
```

Arrange your terminal windows so that you can see both the `psql` prompt, and also the `kafka-avro-console-consumer` from the previous step (this should still be running; re-run it if not). 

Now insert a row in the Postgres `movies` table—you should see almost instantly the same data appear in the Kafka topic. 

```sql
INSERT INTO movies(id,title,release_year) VALUES (937,'Top Gun',1986);
```

## Exercise 3: Your Own Schema Design

1. Break into groups of two to four people. In your groups, discuss some simple business problems within each person's domain. 

2. Agree on one business problem to model. Draw a simple entity diagram of it, and make a list of operations your application must perform on the model. For example, if your business is retail, you might make a simple model of inventory, with entities for location, item, and supplier, plus operations for receiving, transferring, selling, and analysis.

3. Sketch out a simple application to provide a front end and necessary APIs for the system. For each service in the application, indicate what interface it provides (web front-end, HTTP API, etc.) and what computation it does over the data.

4. Some of the entities in your model are truly static, and some are not. Revisit your entity model and decide which entities are should be streams and which are truly tables.

5. Re-draw your diagram from step three with the appropriate tables replaced by streams. For each service in the application, keep its interface constant, but re-consider what computation it does over the data.

6. Time permitting, present your final architecture to the class. Explain how you adjudicated each stream/table duality and what streaming computations you planned.

## Exercise 4: Enriching Data with KSQL

0. Clean up the topic you created in exercise 2 as follows:
```
$ docker-compose exec kafka1 bash
root@kafka1:/# kafka-topics --zookeeper zookeeper:2181 --delete --topic movies-raw
```

1. In separate terminal tabs, keep the worker container from exercise #1 running, and run a second container with the KSQL Server in it:
```
$ docker-compose run ksql
```

Wait for KSQL to finish starting up.

1. In yet another terminal tab (you are a programmer, after all), start up the KSQL CLI like this:
```
$ docker-compose exec ksql bash
root@8ef27b1b86a4:/# ksql
```

2. In the worker container, see the movies data with `head -n 1 streams-demo/data/movies-json.js  | kafkacat -b kafka1:9092 -t movies-raw -P`


3. In the KSQL container, create a stream around the raw movie data: `CREATE STREAM movies_src (movie_id LONG, title VARCHAR, release_year INT) WITH (VALUE_FORMAT='JSON', KAFKA_TOPIC='movies-raw');`

4. As you can see by selecting the records from that stream, the key is null. Re-key it with `CREATE STREAM movies_rekeyed AS SELECT * FROM movies_src PARTITION BY movie_id;`

5. Run a non-persistent select on `movies_rekeyed` in the KSQL window, then stream ten more movie records into the `movies-raw` topic. Watch them appear in the rekeyed stream. 

6. Turn the movies into a table of reference data with `CREATE TABLE movies (movie_id LONG, title VARCHAR, release_year INT) WITH (VALUE_FORMAT='JSON', KAFKA_TOPIC='MOVIES_REKEYED', KEY='movie_id');`

7. In a new worker container (another terminal tab with `docker-compose run worker bash`), do the following:
```
cd streams-demo
./gradlew streamJsonRatings
```

8. Create a stream to represent the ratings: `CREATE STREAM ratings (movie_id LONG, rating DOUBLE) WITH (VALUE_FORMAT='JSON', KAFKA_TOPIC='ratings');`

9. Attempt to join ratings to the movie data with `SELECT m.title, m.release_year, r.rating FROM ratings r LEFT OUTER JOIN movies m on r.movie_id = m.movie_id;`. Note the nulls! We need more movies in the reference stream.

10. `cat` the rest of the `movies-json.js` file into the stream. Notice that the join starts working!

11. Create a table containing average ratings as follows: `CREATE TABLE movie_ratings AS SELECT m.title, SUM(r.rating)/COUNT(r.rating) AS avg_rating, COUNT(r.rating) AS num_ratings FROM ratings r LEFT OUTER JOIN movies m ON m.movie_id = r.movie_id GROUP BY m.title;`

12. Select from that table and inspect the average ratings. Do you agree with them? Discuss. (If you want the table to stop updating, kill the Gradle task that is streaming the ratings—it's been going this whole time.)

### Extra Credit

Rewrite the KSQL queries to use the Avro topic you created in the Connect exercise.
