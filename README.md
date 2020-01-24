# Streams and Tables in Apache Kafka

## Events, streams, and tables

An event records the fact that “something happened” in the world. Conceptually, an event has a key, value, and timestamp. A concrete event could be a plain notification without any additional information, but it could also include the full details of what exactly happened to facilitate subsequent processing. For instance:

- Event key: “Alice”
- Event value: “Is currently in Rome”
- Event timestamp: “Dec. 3, 2019 at 9:06 a.m.”

Other example events include:

- A good was sold
- A row was updated in a database table
- A wind turbine sensor measured 14 revolutions per minute
- An action occurred in a video game, such as “White moved the e2 pawn to e4”
- A payment of $200 was made by Frank to Sally on Nov. 24, 2019, at 5:11 p.m.

Events are captured by an event streaming platform into event streams. An event stream records the history of what has happened in the world as a sequence of events. An example stream is a sales ledger or the sequence of moves in a chess match.

Compared to an event stream, a table represents the state of the world at a particular point in time, typically “now.” An example table is total sales or the current state of the board in a chess match

Streams and tables in Kafka differ in a few ways, notably with regard to whether their contents can be changed, i.e., whether they are mutable.

- A stream provides immutable data. It supports only inserting (appending) new events, whereas existing events cannot be changed. Streams are persistent, durable, and fault tolerant.

- A table provides mutable data. New events—rows—can be inserted, and existing rows can be updated and deleted.Here, an event’s key aka row key identifies which row is being mutated. Like streams, tables are persistent, durable, and fault tolerant. Today, a table behaves much like an RDBMS materialized view because it is being changed automatically as soon as any of its input streams or tables change, rather than letting you directly run insert, update, or delete operations against it.

|                                           | Stream   | Table  |
| ----------------------------------------- |:--------:| ------:|
| First event with key bob arrives          | Insert   | Insert |
| Another event with key bob arrives        | Insert   | Insert |
| Event with key bob and value null arrives | Insert   | Insert |
| Event with key null arrives	            | Insert   | Insert |

## Stream-table duality
we can observe that there is a close relationship between a stream and a table. We call this the stream-table duality. What this means is:

- We can turn a stream into a table by aggregating the stream with operations such as COUNT() or SUM(), for example. In our chess analogy, we could reconstruct the board’s latest state (table) by replaying all recorded moves (stream).
- We can turn a table into a stream by capturing the changes made to the table—inserts, updates, and deletes—into a “change stream.” This process is often called change data capture or CDC for short. In the chess analogy, we could achieve this by observing the last played move and recording it (into the stream) or, alternatively, by comparing the board’s state (table) before and after the last move and then recording the difference of what changed (into the stream), though this is likely slower than the first option.

# Topics, Partitions, and Storage Fundamentals

## Kafka Topics
- Conceptually, a topic is an unbounded sequence of serialized events, where each event is represented as an encoded key-value pair or “message.”
- The machines that store and serve the data are called *Kafka brokers*, which are the server component of Kafka

## Storage formats: Serialization and deserialization of events
- Events are serialized when they are written to a topic and deserialized when they are read
- these operations are done solely by the Kafka clients
- Common serialization formats used by Kafka clients include Apache Avro™, (with the Confluent Schema Registry), Protobuf, and JSON.
https://docs.confluent.io/current/schema-registry/index.html?_ga=2.217009494.13037944.1579219029-874413728.1579219029

- Kafka brokers, on the other hand, are agnostic to the serialization format or “type” of a stored event. All they see is a pair of raw bytes for event key and event value (<byte[], byte[]> in Java notation) coming in when being written, and going out when being read.
- Brokers not involve serialization/deserialization of data

## Storage is partitioned
- Kafka topics are partitioned, meaning a topic is spread over a number of “buckets” located on different brokers. This distributed placement of your data is very important for scalability because it allows client applications to read the data from many brokers at the same time.

- When creating a topic, you must choose the number of partitions it should contain. Each partition then contains one specific subset of the full data in a topic (see partitioning in databases and partitioning of a set). To make your data fault tolerant, every partition can be replicated, even across geo-regions or datacenters, so that there are always multiple brokers that have a copy of the data just in case things go wrong, you want to do maintenance on the brokers, and so on. A common setting in production is a replication factor of 3 for a total of three copies.

## Event producers determine event partitioning
Producers determine event partitioning—how events will be spread over the various partitions in a topic. More specifically, they use a partitioning function ƒ(event.key, event.value) to decide which partition of a topic an event is being sent to. The default partitioning function is ƒ(event.key, event.value) = hash(event.key) % numTopicPartitions so that, in most cases, events will be spread evenly across the available topic partitions (we will later discuss what happens when this is not the case). The partitioning function actually provides you with further information in addition to the event key for determining the desired target partition, such as the topic name and cluster metadata.

## How to partition your events: Same event key to same partition
The primary goal of partitioning is the ordering of events: producers should send “related” events to the same partition because Kafka guarantees the ordering of events only within a given partition of a topic—not across partitions of the same topic.

To give an example of how to partition a topic, consider producers that publish geo-location updates of trucks for a logistics company. In this scenario, any events about the same truck should always be sent to one and the same partition. This can be achieved by picking a unique identifier for each truck as the event key (e.g., its licensing plate or vehicle identification number), in combination with the default partitioning function.

However, there is another reason why partitioning matters. Stream processing applications typically operate in so-called Kafka consumer groups that all read from the same topic(s) for collaborative processing of data in parallel. In such cases, it’s important to be able to control which partitions go to different participants within the same group.

So, what are the most common reasons why events with the same event key may end up in different partitions? Two causes stand out:

1) *Topic configuration*: someone increased the number of partitions of a topic. In this scenario, the default partitioning function ƒ(event.key, event.value) now assigns different target partitions for at least some of the events because the modulo parameter has changed.
2) *Producer configuration*: a producer uses a custom partitioning function.

My tip: if in doubt, use 30 partitions per topic. This is a good number because (a) it is high enough to cover some really high-throughput requirements, (b) it is low enough that you will not hit the limit anytime soon of how many partitions a single broker can handle, even if you create many topics in your Kafka cluster, and (c) it is a highly composite number as it is evenly divisible by 1, 2, 3, 5, 6, 10, 15, and 30. This benefits the processing layer because it results in a more even workload distribution across application instances when horizontally scaling out (adding app instances) and scaling in (removing instances). Since Kafka supports hundreds of thousands of partitions in a cluster, this over-partitioning strategy is a safe approach for most users.

# From storage to processing
Topics live in Kafka’s storage layer—they are part of the Kafka “filesystem” powered by the brokers. In contrast, streams and tables are concepts of Kafka’s processing layer, used in tools like ksqlDB and Kafka Streams. These tools process your events stored in “raw” topics by turning them into streams and tables—a process that is conceptually very similar to how a relational database turns the bytes in files on disk into an RDBMS table for you to work with.

An **event stream** in Kafka is a topic with a schema. Keys and values of events are no longer opaque byte arrays but have specific types, so we know what’s in the data. Like a topic, a stream is unbounded.

The following examples use the Java notation of `<eventKey, eventValue>` for the data types of event key and value, respectively. For instance, `<byte[], String>` means that the event key is an array of raw bytes, and the event value is a String, i.e., textual data like a `VARCHAR`. More complex data types can be used, such as an Apache Avro™ schema that defines a `GeoLocation` type.

**Example #1**: A `<byte[], byte[]>` topic is read and deserialized by a consumer client as a `<String, String>` stream of geolocation events. Or, we could use a better type setup and read it as a `<User, GeoLocation>` stream, which is what I’d prefer.

Here are code examples for how to read a topic as a stream:

- ksqlDB:
```
-- Create ksqlDB stream from Kafka topic.
CREATE STREAM myStream (username VARCHAR, location VARCHAR)
  WITH (KAFKA_TOPIC='input-topic', VALUE_FORMAT='...');
```
- Kafka Streams:
```
// Create KStream from Kafka topic.
StreamsBuilder builder = new StreamsBuilder();
KStream<String, String> stream =
  builder.stream("input-topic", Consumed.with(Serdes.String(), Serdes.String()));
```