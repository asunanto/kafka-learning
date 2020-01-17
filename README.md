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