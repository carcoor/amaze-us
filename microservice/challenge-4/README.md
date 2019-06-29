
  

# Challenge 4 - Mysteries of the Cats
Discovery is upon us! The mystery has to be solved!
After years of research, century of mysticism, we have it.
Here is the truth, we can say with certainty that cats have 7 moods :
```scala
  val GROWL         = "grr"
  val HISS          = "kssss"
  val PURR          = "rrrrr"
  val THROWGLASS    = "cling bling"
  val ROLLONFLOOR   = "fffff"
  val SCRATCHCHAIRS = "gratgrat"
  val LOOKDEEPINEYES = "-o-o-___--"
``` 
Not counting of course, there default "Miaw" mood.
We are on mission. So should you!
First, let's modelize our cats in Scala.
To match reality, those cats should randomly change mood every 27 seconds.
Now, let's simulate a thousand of those during a day.
We will need then to store all of those different cats mood.

Once that's done, we will at last be able to generate some statistics: average, median, variance of the different mood
over time, anything that can help us pierce the secret.
This of course should scale orthogonally to the number of cats we are simulating, we don't want to end up with angry
cats, right ?

Make sparks, solve the mystery!

## Expected steps

+ Create a branch of this project (or fork it in your github account if you prefer)
+ Do you **_thang_** inside this folder (challenge-4)
+ Push your change inside a Pull Request to our master

## Architecture
  ![
](http://www.antaki.ca/cats/MysteriesOfTheCats.png)
  
### Tech choices
- I used Java because it's the language i'm the best at and it really works well for this problem. I don't have experience with Scala
Why Java 11 and not 12 it's because the Kafka release notes don't say Java 12 support and I didn't want to risk.
- I chose SpringBoot because it's the easiest, fastest framework to develop in.
- The cat-mood-producer uses Java 8 Parallel Stream, it can also use the number of provided threads to have more control about the number of thread, i found the Parallel Stream to be faster so it's the default implementation -Dexecutor.implementation=stream|anything
- Kafka was a no-brainer because of its scalability and its stability and maturity.
- The 2 apps can be monitored with JMX using VisualVM for ex, I used Codahale (dropwizard) metrics
- The use of Timescaledb was because it had good performance and i was familiar with SQL (haven't used PostgreSQL before), I could have used InfluxDB. Moreover there was a mention of storage, SQL provides a good long term storage.
- I used Grafana because it's Open Source and popular
- I didn't optimize the JVM: Xmx, Xms, default, max Metaspace size. Starting from Java 9 the default GC algorithm is G1. It's always a good habit to set -XX:MaxMetaspaceSize as it's unlimited by default

## Prerequisites
1. Install OpenJDK 11
2. Install Maven 3.6

3. Install kafka_2.12-2.1.1

4. Install Docker

5. Start zookeeper
```
sudo {KAFKA_INSTALL_DIR}bin/zookeeper-server-start.sh config/zookeeper.properties
```

6. Start kafka server

```
sudo {KAFKA_INSTALL_DIR}/bin/kafka-server-start.sh config/server.properties
```

7. Create cats topic: we're using replication factor 1 as we only have 1 broker, in PROD we would have 3 or more brokers so we can increase it to 2 or 3 or more depending.
```
sudo {KAFKA_INSTALL_DIR}/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 50 --topic cats
```

8. Run grafana
```
docker run -d --name=grafana -p 3000:3000 grafana/grafana
```
9. Run PostgreSQL with timescaledb
```
docker run -d --name timescaledb -p 5432:5432 -e POSTGRES_PASSWORD=password timescale/timescaledb:latest-pg11
```

## Building the projects

1. Start the producer
```
cd microservice/challenge-4/cat-mood-producer
mvn clean package
```

2. Start the consumer
```
cd microservice/challenge-4/mood-consumer
mvn clean package
```

## Running the projects

1. Start the producer:
```
cd microservice/challenge-4/cat-mood-producer
java -jar target/cat-mood-producer-0.0.1-SNAPSHOT.jar
```

For more control:
```
java -Dlogging.level.root={LOG_LEVEL} -Dcats={NB_CATS} -Dchange.mood.interval.seconds={SECONDS} -jar target/cat-mood-producer-0.0.1-SNAPSHOT.jar
```
or
```
mvn spring-boot:run
```
  
2. Start the consumer:
```
cd microservice/challenge-4/mood-consumer
java -jar target/cat-mood-consumer-0.0.1-SNAPSHOT.jar
```

For more control:
```
java -Dlogging.level.root={LOG_LEVEL} -Ddb.batch.size={BATCH_SIZE} -Dconsumer.threads={NB_THREADS} -Dkafka.server.address={SERVER_ADDRESS} -Dkafka.group.id={GROUP_ID} -jar target/cat-mood-consumer-0.0.1-SNAPSHOT.jar
```
or
```
mvn spring-boot:run
```

3. Integrate PostgreSQL with Grafana

Go to http://localhost:3000 and add the PostgreSQL data source

```
Host: host.docker.internal:5432

Database: postgres

User: postgres

SSL Mode: disable

TimescaleDB: enable
```
4. Add the dashboard with the following SQL queries:

For the average:

```sql

SELECT cm.time, AVG(mood)

OVER(ORDER BY  time  ROWS  BETWEEN  29  PRECEDING  AND CURRENT ROW) AS avg_mood

FROM cats_mood cm

WHERE

$__timeFilter(time)

```
For the variance:

```sql

SELECT cm.time, variance(mood)

OVER(ORDER BY  time  ROWS  BETWEEN  29  PRECEDING  AND CURRENT ROW) AS variance_mood

FROM cats_mood cm

WHERE

$__timeFilter(time)

```

For the median:
```sql
SELECT cm.time,
       percentile_disc(0.5) within group (order by mood) as median
   
           
FROM cats_mood cm

WHERE

  $__timeFilter(time)
  
group by cm.time

```

## Tests

### Functional Test

I generated 200,000 moods using the cat-mood-producer then I ran the cat-mood-consumer and then I checked the database to make sure we have 200,000 records.

We could have an integration test that does that.

  

### Demo

  
  Running cat-mood-producer
![
](http://www.antaki.ca/cats/running_cat_mood_producer.png)

Running cat-mood-consumer
![Running cat-mood-producer](http://www.antaki.ca/cats/running_cat_mood_consumer.png)


Grafana stats
![
](http://www.antaki.ca/cats/grafana_new.png)
### Benchmark

| Nb cats                    | Producer | Consumer 10 threads | Consumer 20 threads | Consumption rate (per second) | Comment                                                              |
|----------------------------|----------|---------------------|---------------------|-------------------------------|----------------------------------------------------------------------|
| 5000 = 1000 every 5s       | 392 ms   | 800 ms              | 1.1s                | 6,250                         |                                                                      |
| 250,000 = 50,000 every 5s  | 808 ms   | 13s                 | 10s                 | 19,230                        |                                                                      |
| 1,000,000= 200,000 every 5 | 2469     | 37s                 | 37s                 | 27,027                        | This shows bottleneck is from DB side  as it's running on Docker MAC |

I've used 5s instead of 27s for speed :)
## Improvements

Currently using JSON for ease of use, but using Avro or GPB (Google Protocol Buffer) is faster. Moreover we need to use a common model between the 2 projects.

### cat-mood-producer

- Currently creating the cats is done in the same project. In real life this would come from another process that would add the creation event in Kafka and we would subscribe to it.
- The single process producer is not resilient, i could have many processes instead of 1.
- The scheduling is not scalable at the moment as it's done with only 1 thread in the same application, we can use ShedLock or Quartz in order to spread the work to be done over a cluster of servers
- Add unit/integration tests
- Make executable smaller:
- Maybe investigate a lighter weight microservice framework such as [https://quarkus.io/](https://quarkus.io/)
- Use the Java module System in order to build a smaller JRE with jlink and Graalvm to build a faster native executable
- Deploy using a Docker image: I added a Dockerfile i can build and run but I have an issue connecting to Kafka on my host: I don't have time to fix it.

### cat-mood-consumer
- Currently we run only 1 consumer (1 process with n threads), we could run more than 1 consumer and have each consumer handle specific partitions
- Add unit/integration tests
- Use the Java module System in order to build a smaller JRE with jlink and Graalvm to build a faster native executable
- Deploy using a Docker image

### kafka
- We currently have a topic with 50 partitions. If we need more scalability we can increase that.
- Tried using spotify kafka Docker image but it was running an older version so reverted to locally installed Kafka