# Apache Kafka with Node.js and Python

A simple-to-advanced Kafka guide for backend development and interviews.

Examples use:

- Node.js 20+ with [KafkaJS](https://kafka.js.org/)
- Python 3.10+ with `confluent-kafka`
- Apache Kafka 3.x concepts

Kafka is a distributed event streaming platform. It stores events durably, lets applications publish and read them, and scales by partitioning data across brokers.

---

# 1. Kafka in simple words

Imagine a durable, distributed log:

```text
Producer -> Topic -> Partitions -> Consumer group -> Application
```

A producer writes an event to Kafka. Kafka stores the event in a topic partition. A consumer reads it later using an offset. The event stays in Kafka according to the topic retention policy, so multiple independent applications can read the same event.

Example event:

```json
{
  "eventId": "evt_1001",
  "type": "order.created",
  "orderId": "ord_42",
  "userId": "usr_7",
  "createdAt": "2026-09-25T10:00:00.000Z"
}
```

Kafka is commonly used for:

- Event-driven microservices
- Activity and audit logs
- Data pipelines
- Log collection
- Metrics and telemetry
- Notifications
- Payment and order workflows
- Change Data Capture
- Stream processing
- Integration between independent systems

Kafka is not automatically the right choice for every job. A normal request-response API, a small task queue, or a transactional database operation may be simpler with another tool.

---

# 2. Core Kafka terminology

## Event or record

A record is one message stored by Kafka. It can contain:

- Key
- Value
- Timestamp
- Headers
- Offset
- Partition number

Example:

```text
key:   user-42
value: {"type":"user.updated","userId":"42"}
```

Kafka does not require JSON. Values can be JSON, text, Avro, Protobuf, bytes, or another format.

## Topic

A topic is a named stream of records, such as:

```text
orders
payments
user-events
inventory-updates
```

A topic is logical. Its data is physically distributed across partitions.

## Partition

A topic is divided into one or more partitions. Each partition is an append-only ordered log.

```text
orders topic
  partition 0: record 0 -> record 1 -> record 2
  partition 1: record 0 -> record 1 -> record 2
  partition 2: record 0 -> record 1 -> record 2
```

Important rules:

- Ordering is guaranteed inside one partition, not across the whole topic.
- A partition is the unit of parallelism.
- A record belongs to exactly one partition.
- A partition has a monotonically increasing offset.
- A partition can be replicated to multiple brokers.

## Offset

An offset is the position of a record within one partition. Offsets are not globally unique.

```text
partition 0: offset 0, offset 1, offset 2
partition 1: offset 0, offset 1, offset 2
```

An offset is not normally a permanent global ID. Records can expire, so offset `100` later may refer to no available record.

## Broker

A broker is a Kafka server. It receives writes, stores partition data, serves reads, and participates in cluster coordination.

A Kafka cluster usually contains multiple brokers:

```text
Broker 1
Broker 2
Broker 3
```

## Kafka cluster

A cluster is a group of brokers working together. A cluster has controllers that manage metadata such as broker membership and partition leadership.

## Controller

The controller manages cluster metadata and administrative actions such as:

- Broker membership
- Partition leadership
- Replica assignment
- Failure handling
- Topic creation and configuration

Modern Kafka uses KRaft mode. Older deployments used ZooKeeper for metadata management.

## KRaft

KRaft is Kafka's built-in metadata consensus mode. It removes the separate ZooKeeper dependency.

Terms you may see:

- Controller quorum: controller nodes that maintain Kafka metadata.
- Controller node: a broker that also participates in the controller quorum.
- Combined mode: one process acts as both broker and controller, common for development.
- Dedicated controller mode: controllers and brokers are separated, common for larger production clusters.

## Leader replica

Each partition has one leader replica. Producers and consumers normally communicate with the partition leader.

## Follower replica

Follower replicas copy the leader's data. A follower can become leader if the current leader fails and the cluster considers it sufficiently caught up.

## Replication factor

Replication factor is the number of copies of each partition.

```text
replication.factor = 3
```

A replication factor of 3 means one leader and two followers, usually on different brokers.

Replication protects availability, not accidental deletion or application bugs. Backups and recovery plans are still needed.

## ISR: in-sync replicas

ISR means In-Sync Replicas. These are replicas considered caught up enough to become leaders.

If a follower is slow or disconnected, it may leave the ISR. A partition with fewer ISR members has less failure tolerance.

Important settings:

```properties
min.insync.replicas=2
acks=all
```

Together, these can require at least two in-sync replicas to acknowledge a write.

## Preferred replica

The preferred replica is the first replica in a partition's replica assignment. Kafka can rebalance leadership back to preferred replicas to avoid uneven leader distribution.

## Partition reassignment

Partition reassignment moves replicas between brokers. It is used when:

- Adding brokers
- Removing brokers
- Fixing uneven distribution
- Changing replication layout

Perform reassignment gradually. Large movements consume network and disk bandwidth.

## Log

A Kafka partition is an append-only log. New records are appended; records are not normally updated in place.

## Segment

A partition log is split into segment files. Kafka rolls segments based on size or time. Older segments can be deleted according to retention rules.

## Retention

Retention controls how long or how much data Kafka keeps.

Time-based retention:

```properties
retention.ms=604800000
```

Size-based retention:

```properties
retention.bytes=10737418240
```

Kafka can retain records even after every consumer has read them. Reading does not delete a record.

## Log compaction

Compaction keeps the latest record for each key, eventually removing older records with the same key.

```properties
cleanup.policy=compact
```

Compaction is useful for state topics, such as the latest profile for each user. It is not an exact audit log because old values may eventually disappear.

Tombstone record:

```text
key: user-42
value: null
```

A tombstone indicates deletion in a compacted topic. Tombstones remain for a configured period before being cleaned up.

## Topic configuration

Common topic settings:

| Setting | Purpose |
|---|---|
| `partitions` | Number of partitions |
| `replication.factor` | Number of replicas |
| `retention.ms` | Time to retain records |
| `retention.bytes` | Size limit per partition |
| `cleanup.policy` | `delete`, `compact`, or both |
| `min.insync.replicas` | Minimum ISR for acknowledged writes |
| `max.message.bytes` | Maximum record size |
| `segment.bytes` | Segment file size |
| `compression.type` | `none`, `gzip`, `snappy`, `lz4`, or `zstd` |

## Producer

A producer publishes records to topics.

## Consumer

A consumer reads records from topics and processes them.

## Consumer group

A consumer group is a set of consumers cooperating to process a topic. Kafka assigns each partition to at most one consumer within a group.

Example:

```text
Topic has 4 partitions
Consumer group has 2 consumers

Consumer A -> partitions 0 and 1
Consumer B -> partitions 2 and 3
```

A group cannot actively use more consumers than the number of partitions. Extra consumers remain idle.

Different consumer groups each receive the full stream independently:

```text
orders topic
  billing-group       -> billing service
  analytics-group     -> analytics service
  notification-group -> notification service
```

## Group coordinator

The group coordinator is the broker responsible for managing a consumer group. It tracks membership, heartbeats, assignments, and committed offsets.

## Rebalance

A rebalance redistributes partitions among consumers in a group. It occurs when:

- A consumer joins
- A consumer leaves
- A consumer crashes
- Topic partitions change
- Subscription changes

Rebalances temporarily affect processing. Long processing inside `poll` or missing heartbeats can cause a consumer to be considered dead.

## Group membership

Consumers join a group using `group.id`. Each member sends heartbeats and receives partition assignments.

## Partition assignment strategies

Common assignment strategies include:

- Range assignor
- Round-robin assignor
- Sticky assignor
- Cooperative sticky assignor

Cooperative assignment can reduce disruptive stop-the-world rebalances.

## Consumer offset commit

A consumer commits the next offset it plans to read. Committed offsets are stored in Kafka's internal `__consumer_offsets` topic.

Commit styles:

- Auto commit: client periodically commits.
- Manual synchronous commit: wait for broker confirmation.
- Manual asynchronous commit: do not wait immediately.
- Commit after processing: usually at-least-once behavior.
- Commit before processing: can lose messages when the process fails.

## `auto.offset.reset`

Controls where a group starts when it has no committed offset or its offset is unavailable:

- `earliest`: start from the oldest retained record.
- `latest`: start from new records.
- `none`: throw an error when no offset exists.

This setting does not reset an existing valid committed offset.

## At-most-once

Commit before processing. A crash after commit and before processing can lose the record, but a record is not normally processed twice.

## At-least-once

Process first, then commit. A crash after processing but before commit can cause the record to be processed again.

This is the common practical default. Make handlers idempotent.

## Exactly-once semantics

Exactly-once processing means a record's effect is committed exactly once within a supported transactional design. It is not a magic property that makes every external side effect exactly once.

Kafka transactions can atomically write output records and commit consumed offsets. A database or email provider outside Kafka still needs its own integration strategy.

## Idempotency

An idempotent handler produces the same final result when it receives the same event more than once.

Common techniques:

- Store an event ID in a processed-events table with a unique constraint.
- Use an upsert keyed by event ID.
- Use a business key such as `orderId` and event version.
- Make state transitions conditional.
- Use an idempotency key for external APIs.

## Producer key

The key influences partition selection. Records with the same key are normally sent to the same partition, preserving per-key order.

```text
key = order-42
```

Good keys are:

- Stable
- Present for records that need ordering
- Distributed enough to prevent hot partitions

A bad key can send almost all records to one partition.

## Null key

Without a key, the producer uses the configured partitioner. There is no per-entity ordering guarantee for records without a stable key.

## Headers

Headers are small key-value metadata attached to a record:

```text
event-type: order.created
trace-id: 7f4a
schema-version: 2
```

Headers are useful for routing and tracing, but do not put large payloads in them.

## Producer acknowledgement: `acks`

`acks` controls how much broker acknowledgement the producer waits for:

- `acks=0`: do not wait for a broker response; fastest, weakest durability.
- `acks=1`: leader acknowledges; a leader failure before replication can lose data.
- `acks=all` or `-1`: all required in-sync replicas acknowledge; strongest normal durability.

## Producer retries

A producer can retry transient failures. Retries can create duplicates if the first request succeeded but the response was lost.

Use idempotent producer mode where supported:

```properties
enable.idempotence=true
```

Idempotence prevents duplicate writes caused by producer retries within the producer's supported session.

## Delivery timeout and request timeout

- `request.timeout.ms`: how long the producer waits for a broker response.
- `delivery.timeout.ms`: total time allowed for delivery including retries.

Set them consistently with the service's latency and retry requirements.

## Batching

The producer can group records into batches. Larger batches can improve throughput but add latency.

Important settings:

- `batch.size`
- `linger.ms`
- `compression.type`

## Compression

Compression reduces network and storage usage. `lz4` and `zstd` are common choices. Compression effectiveness depends on the data.

## Consumer poll loop

Consumers repeatedly fetch records. The client must continue polling or heartbeating within configured limits.

Long processing should be handled by:

- Increasing the allowed processing interval carefully.
- Pausing partitions while work completes.
- Sending work to a worker pool.
- Using smaller batches.
- Splitting a slow workload into another topic.

## Fetch settings

Common consumer fetch settings:

- `fetch.min.bytes`: minimum data before broker responds.
- `fetch.max.wait.ms`: maximum wait to build a fetch response.
- `max.partition.fetch.bytes`: maximum data per partition per fetch.
- `max.poll.records`: maximum records returned per poll in clients that support it.

## Backpressure

Backpressure happens when consumers cannot process records as fast as producers create them.

Signs:

- Consumer lag grows.
- Processing latency increases.
- Memory usage grows.
- Retry topics fill.

Solutions:

- Add partitions and consumers.
- Increase processing parallelism safely.
- Batch database writes.
- Reduce per-record work.
- Add rate limits upstream.
- Use pause and resume.
- Move slow work to a worker queue.

## Consumer lag

Consumer lag measures how far a consumer's committed or current position is behind the log end offset.

Conceptually:

```text
lag = log end offset - consumer position
```

Lag is not automatically an error. A short spike may be normal. Sustained or rapidly growing lag needs investigation.

## Dead letter topic

A dead letter topic, or DLT/DLQ, stores records that cannot be processed after a defined retry policy.

A good DLT record includes:

- Original topic
- Original partition
- Original offset
- Error type and message
- Retry count
- Failed timestamp
- Original key and value
- Trace or correlation ID

Never silently discard poison messages.

## Retry topic

A retry topic delays a failed record before another attempt. Common patterns use separate topics such as:

```text
orders.retry.1m
orders.retry.10m
orders.dlt
```

Avoid immediate infinite retry loops that block healthy records.

## Poison pill

A poison pill is a record that repeatedly causes consumer failure, often due to invalid data, schema incompatibility, or a bug.

Quarantine it, alert on it, and continue processing other records according to the business policy.

## Tombstone

A tombstone is a record with a key and null value. It represents deletion in a compacted topic.

## Watermark

A watermark is a position related to available records. In Kafka tooling, low and high watermarks often represent the oldest available offset and the next offset that will be written.

## High-water mark

The high-water mark is the offset up to which records are replicated and available for consumers under Kafka's consistency rules. Terminology can differ slightly by client and metric.

## Log end offset

The log end offset is the next offset to be written at the end of a partition. It is commonly used with consumer position to estimate lag.

## Consumer position

The consumer position is the next offset the consumer will fetch.

## Committed position

The committed position is the offset stored for a consumer group. After a restart, a consumer generally resumes from its committed position.

## Bootstrap server

A bootstrap server is an initial broker address used by clients to discover the cluster. The client does not need every broker address in the list.

## Metadata

Metadata includes topics, partitions, leaders, replicas, and broker information. Clients refresh metadata as the cluster changes.

## Advertised listeners

Brokers advertise addresses that clients should use. Incorrect advertised listeners are a common Docker and Kubernetes connection problem.

## Rack awareness

Rack awareness places replicas across failure domains such as availability zones or racks. It reduces the chance that one zone failure removes all replicas.

## Quota

Kafka quotas limit client or user resource usage, commonly network bandwidth and request rates. Quotas help prevent one client from harming the cluster.

## ACL

An Access Control List defines which principals can perform operations on resources such as topics and consumer groups.

Examples:

- Read from `orders`
- Write to `payments`
- Describe a topic
- Create a topic
- Join a consumer group

## Principal

A principal is an authenticated identity such as a user, service account, or certificate identity.

## SASL

SASL is an authentication framework. Common Kafka mechanisms include:

- `SASL/PLAIN`: username and password; use with TLS.
- `SASL/SCRAM`: salted challenge-response credentials.
- `SASL/OAUTHBEARER`: OAuth-style tokens.
- `GSSAPI`: Kerberos, common in some enterprise environments.

## TLS / SSL

TLS encrypts data in transit and can authenticate brokers and clients. Kafka configuration often uses `ssl.*` names even when TLS is the modern term.

## Schema

A schema defines the structure and types of an event. It prevents producers and consumers from guessing different shapes.

## Schema Registry

A Schema Registry stores and versions schemas, commonly with Avro, JSON Schema, or Protobuf. It can enforce compatibility rules.

Common compatibility modes:

- Backward: new consumers can read old data.
- Forward: old consumers can read new data.
- Full: both backward and forward compatible.
- None: no compatibility enforcement.

Compatibility strategy:

- Add optional fields with defaults.
- Do not rename or remove fields casually.
- Version breaking changes.
- Keep event contracts documented.

## Avro

Avro is a compact binary serialization format commonly paired with Schema Registry. It stores a schema separately or references a schema ID in the message.

## Protobuf

Protocol Buffers are strongly typed binary messages. Fields have numeric tags, and compatible evolution requires care with field numbers.

## JSON Schema

JSON Schema describes JSON structures. It is readable and easy to integrate, though often larger than binary formats.

## Consumer-driven contract

Consumers document the fields and behavior they need. Producer changes are tested against those expectations.

## Kafka Connect

Kafka Connect is a framework for moving data between Kafka and external systems.

Examples:

- PostgreSQL to Kafka using a source connector
- Kafka to Elasticsearch using a sink connector
- Kafka to object storage

Terms:

- Source connector: external system to Kafka.
- Sink connector: Kafka to external system.
- Worker: process that runs connectors and tasks.
- Task: unit of connector work.
- Standalone mode: simple, single-process configuration.
- Distributed mode: scalable, fault-tolerant configuration.
- Converter: serializes connector data.
- SMT: Single Message Transform, a lightweight record transformation.

## Change Data Capture

CDC captures database changes and publishes them as events. Debezium is a common CDC tool used with Kafka Connect.

CDC concerns:

- Snapshot behavior
- Ordering per table or key
- Schema changes
- Deletes and tombstones
- Transaction boundaries
- Reprocessing and duplicate events

## Kafka Streams

Kafka Streams is a Java library for building stream-processing applications. It supports:

- Stateless transformations
- Stateful transformations
- Joins
- Aggregations
- Windowing
- Exactly-once processing modes

Node.js and Python applications commonly use Kafka as producers and consumers, while Kafka Streams is most mature in the JVM ecosystem.

## ksqlDB

ksqlDB lets teams process Kafka topics using SQL-like statements. It can be useful for simple stream transformations and materialized views.

## Event sourcing

Event sourcing stores state changes as an append-only event history. Current state is derived by replaying events.

Kafka can be part of an event-sourcing architecture, but a Kafka topic is not automatically a complete event-sourcing solution. Retention, snapshots, schema evolution, replay behavior, and audit requirements must be designed.

## CQRS

Command Query Responsibility Segregation separates writes from reads. Kafka can distribute domain events from the command side to build read models.

## Outbox pattern

The outbox pattern solves the dual-write problem: updating a database and publishing an event reliably.

```text
Transaction:
  update business table
  insert event into outbox table

Outbox publisher:
  read unpublished outbox rows
  publish to Kafka
  mark rows published
```

The publisher may publish duplicates, so consumers must be idempotent. CDC can also publish the outbox table.

## Dual-write problem

Writing to a database and Kafka separately can produce inconsistent results:

```text
Database succeeds -> Kafka fails
Kafka succeeds    -> Database fails
```

Use an outbox, CDC, or a carefully designed transaction boundary.

## Exactly-once versus effectively-once

A system is often called effectively once when duplicate delivery is possible but idempotency makes the final business state equivalent to one successful application.

This is usually more realistic than promising exactly-once effects across databases and external APIs.

---

# 3. Kafka architecture

## Data flow

```text
Producer
   |
   | key chooses partition
   v
Topic: orders
   | partition 0 -> leader broker 1, follower broker 2
   | partition 1 -> leader broker 2, follower broker 3
   | partition 2 -> leader broker 3, follower broker 1
   |
   +--> billing-group
   +--> analytics-group
```

## Partition ordering

If all events for `order-42` use the key `order-42`, Kafka routes them to the same partition and preserves their order there.

There is no global order across partitions. If you require total ordering, use one partition, but this limits throughput.

## Choosing partition count

Consider:

- Expected write throughput
- Expected read throughput
- Number of consumers needed
- Key distribution
- Future growth
- Broker count and storage

You can increase partitions later, but changing the count can change key-to-partition mapping and affect ordering. Plan carefully before increasing them.

Approximate consumer parallelism:

```text
maximum active consumers in one group = number of partitions
```

## Hot partitions

A hot partition receives disproportionate traffic, often because of a popular key or a null/poor partitioning strategy.

Solutions:

- Improve the key distribution.
- Use a composite key where strict per-entity ordering is not required.
- Split a very hot entity into shards.
- Increase partitions only when the partitioner can distribute traffic.

Do not change keys casually if consumers depend on per-key ordering.

## Replication and durability

A strong common production baseline is:

```properties
replication.factor=3
min.insync.replicas=2
acks=all
enable.idempotence=true
```

These are not universal defaults. Choose them based on durability, availability, latency, and cost requirements.

## Failure example

With replication factor 3:

```text
Broker 1: leader
Broker 2: follower
Broker 3: follower
```

If Broker 1 fails, Kafka can elect an in-sync follower. If followers are not caught up, the cluster may reject writes rather than violate the configured durability guarantee.

## Availability versus consistency

`unclean.leader.election.enable=true` can elect a replica that is not in the ISR. This may restore availability but can lose acknowledged records. Understand the tradeoff before enabling it.

## Storage and disk

Kafka depends heavily on disk throughput and page cache. Monitor:

- Disk utilization
- Disk latency
- Log directory failures
- Segment count
- Retention growth
- Network throughput

Use fast local disks or carefully designed persistent storage. Avoid filling disks; Kafka needs room for normal operation and cleanup.

---

# 4. Node.js with KafkaJS

## Install

```bash
npm install kafkajs
```

Set `"type": "module"` in `package.json`.

## Create a Kafka client

```js
import { Kafka, logLevel } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'orders-api',
  brokers: (process.env.KAFKA_BROKERS || 'localhost:9092').split(','),
  ssl: process.env.KAFKA_SSL === 'true',
  sasl: process.env.KAFKA_USERNAME
    ? {
        mechanism: 'scram-sha-512',
        username: process.env.KAFKA_USERNAME,
        password: process.env.KAFKA_PASSWORD
      }
    : undefined,
  logLevel: logLevel.INFO
});
```

For a local unauthenticated broker, `brokers: ['localhost:9092']` is enough.

## Producer example

```js
import { Kafka } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'orders-producer',
  brokers: ['localhost:9092']
});

const producer = kafka.producer({
  allowAutoTopicCreation: false,
  idempotent: true,
  maxInFlightRequests: 5
});

await producer.connect();

try {
  const event = {
    eventId: crypto.randomUUID(),
    type: 'order.created',
    orderId: 'order-42',
    createdAt: new Date().toISOString()
  };

  const result = await producer.send({
    topic: 'orders',
    acks: -1,
    compression: CompressionTypes.GZIP,
    messages: [{
      key: event.orderId,
      value: JSON.stringify(event),
      headers: {
        'event-type': 'order.created',
        'schema-version': '1'
      }
    }]
  });

  console.log(result);
} finally {
  await producer.disconnect();
}
```

The complete import needs `crypto` and `CompressionTypes`:

```js
import crypto from 'node:crypto';
import { CompressionTypes, Kafka } from 'kafkajs';
```

Production producer service:

```js
import crypto from 'node:crypto';
import { CompressionTypes, Kafka } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'orders-api',
  brokers: ['localhost:9092']
});

const producer = kafka.producer({
  idempotent: true,
  maxInFlightRequests: 5,
  allowAutoTopicCreation: false
});

export async function startProducer() {
  await producer.connect();
}

export async function publishOrderCreated(order) {
  const event = {
    eventId: crypto.randomUUID(),
    type: 'order.created',
    version: 1,
    orderId: order.id,
    userId: order.userId,
    createdAt: new Date().toISOString()
  };

  return producer.send({
    topic: 'orders',
    acks: -1,
    compression: CompressionTypes.GZIP,
    messages: [{
      key: order.id,
      value: JSON.stringify(event),
      headers: {
        'event-type': event.type,
        'schema-version': String(event.version)
      }
    }]
  });
}

export async function stopProducer() {
  await producer.disconnect();
}
```

## Consumer example

```js
import { Kafka } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'billing-consumer',
  brokers: ['localhost:9092']
});

const consumer = kafka.consumer({
  groupId: 'billing-service-v1'
});

await consumer.connect();
await consumer.subscribe({
  topic: 'orders',
  fromBeginning: false
});

await consumer.run({
  autoCommit: false,
  eachMessage: async ({ topic, partition, message, heartbeat }) => {
    const event = JSON.parse(message.value.toString());

    console.log({
      topic,
      partition,
      offset: message.offset,
      event
    });

    await createInvoiceIfNotExists(event);

    await heartbeat();
  }
});
```

Important: with `autoCommit: false`, KafkaJS does not automatically commit after each message. A production application must explicitly commit at a safe point or use a deliberate batching strategy.

## Manual commit after processing

```js
await consumer.run({
  autoCommit: false,
  eachMessage: async ({ topic, partition, message }) => {
    const event = JSON.parse(message.value.toString());
    await processEventIdempotently(event);

    await consumer.commitOffsets([{
      topic,
      partition,
      offset: (BigInt(message.offset) + 1n).toString()
    }]);
  }
});
```

The committed offset is the next offset to consume, so commit `message.offset + 1`.

For higher throughput, process `eachBatch` and commit one offset after a successful batch, while ensuring in-flight processing cannot acknowledge data prematurely.

## KafkaJS batch consumer

```js
await consumer.run({
  autoCommit: false,
  eachBatchAutoResolve: false,
  eachBatch: async ({
    batch,
    resolveOffset,
    heartbeat,
    commitOffsetsIfNecessary,
    isRunning,
    isStale
  }) => {
    for (const message of batch.messages) {
      if (!isRunning() || isStale()) {
        break;
      }

      const event = JSON.parse(message.value.toString());
      await processEventIdempotently(event);
      resolveOffset(message.offset);
      await heartbeat();
    }

    await commitOffsetsIfNecessary();
  }
});
```

## Consumer shutdown

```js
async function shutdown(signal) {
  console.log(`${signal} received`);
  await consumer.disconnect();
  process.exit(0);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

In a real service, avoid multiple shutdown calls and add a bounded timeout.

## KafkaJS retry configuration

```js
const kafka = new Kafka({
  clientId: 'orders-api',
  brokers: ['localhost:9092'],
  retry: {
    initialRetryTime: 300,
    retries: 8,
    maxRetryTime: 30000
  },
  connectionTimeout: 10000,
  authenticationTimeout: 10000,
  requestTimeout: 30000
});
```

Retries are for transient failures. They are not a replacement for idempotency or a dead letter strategy.

## KafkaJS transactions

```js
const producer = kafka.producer({ transactionalId: 'billing-worker-1' });
const consumer = kafka.consumer({ groupId: 'billing-service' });

await producer.connect();
await consumer.connect();
await consumer.subscribe({ topic: 'orders' });

await consumer.run({
  autoCommit: false,
  eachBatch: async ({ batch, resolveOffset, heartbeat }) => {
    const transaction = await producer.transaction();

    try {
      for (const message of batch.messages) {
        const event = JSON.parse(message.value.toString());
        const output = await transform(event);

        await transaction.send({
          topic: 'billing-events',
          messages: [{
            key: message.key?.toString(),
            value: JSON.stringify(output)
          }]
        });

        resolveOffset(message.offset);
        await heartbeat();
      }

      await transaction.sendOffsets({
        consumerGroupId: 'billing-service',
        topics: [{
          topic: batch.topic,
          partitions: [{
            partition: batch.partition,
            offset: (BigInt(batch.lastOffset()) + 1n).toString()
          }]
        }]
      });

      await transaction.commit();
    } catch (error) {
      await transaction.abort();
      throw error;
    }
  }
});
```

Use transactions only when the full design supports them. External database writes still require an outbox or idempotent integration.

## KafkaJS admin operations

```js
const admin = kafka.admin();
await admin.connect();

await admin.createTopics({
  topics: [{
    topic: 'orders',
    numPartitions: 6,
    replicationFactor: 3
  }],
  waitForLeaders: true
});

const metadata = await admin.fetchTopicMetadata({ topics: ['orders'] });
console.log(metadata);

await admin.disconnect();
```

Disable automatic topic creation in production and manage topics through reviewed infrastructure or admin tooling.

---

# 5. Python with confluent-kafka

## Install

```bash
python -m pip install confluent-kafka
```

For Schema Registry clients:

```bash
python -m pip install confluent-kafka[schemaregistry]
```

## Producer example

```python
import json
import os
import socket
import uuid
from datetime import datetime, timezone

from confluent_kafka import Producer

producer = Producer({
    "bootstrap.servers": os.getenv("KAFKA_BROKERS", "localhost:9092"),
    "client.id": socket.gethostname(),
    "acks": "all",
    "enable.idempotence": True,
    "compression.type": "lz4",
    "linger.ms": 5,
})


def delivery_report(error, message):
    if error is not None:
        print(f"Delivery failed: {error}")
        return
    print(
        f"Delivered to {message.topic()} "
        f"partition={message.partition()} offset={message.offset()}"
    )


order = {
    "eventId": str(uuid.uuid4()),
    "type": "order.created",
    "orderId": "order-42",
    "createdAt": datetime.now(timezone.utc).isoformat(),
}

producer.produce(
    topic="orders",
    key=order["orderId"],
    value=json.dumps(order).encode("utf-8"),
    headers={
        "event-type": b"order.created",
        "schema-version": b"1",
    },
    on_delivery=delivery_report,
)

producer.flush(10)
```

`produce()` is asynchronous. `flush()` waits for queued messages to finish or the timeout to expire. In a long-running service, call `poll()` regularly to serve delivery callbacks and use bounded flushes during shutdown.

## Python consumer example

```python
import json
import os

from confluent_kafka import Consumer, KafkaError, KafkaException

consumer = Consumer({
    "bootstrap.servers": os.getenv("KAFKA_BROKERS", "localhost:9092"),
    "group.id": "billing-service-v1",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": False,
})

consumer.subscribe(["orders"])

try:
    while True:
        message = consumer.poll(1.0)

        if message is None:
            continue
        if message.error():
            if message.error().code() == KafkaError._PARTITION_EOF:
                continue
            raise KafkaException(message.error())

        event = json.loads(message.value().decode("utf-8"))

        try:
            process_event_idempotently(event)
            consumer.commit(message=message, asynchronous=False)
        except Exception as error:
            print(f"Processing failed at offset {message.offset()}: {error}")
            # Route to a retry topic or DLT before committing, according to policy.
            break
finally:
    consumer.close()
```

The commit occurs only after processing succeeds. A crash between processing and commit can cause a duplicate, so `process_event_idempotently()` is essential.

## Python consumer with rebalance callbacks

```python
from confluent_kafka import Consumer, KafkaException


def on_assign(consumer, partitions):
    print(f"Assigned: {partitions}")
    consumer.assign(partitions)


def on_revoke(consumer, partitions):
    print(f"Revoked: {partitions}")
    # Flush work and commit only offsets that were fully processed.


consumer = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "orders-workers",
    "enable.auto.commit": False,
})
consumer.subscribe(
    ["orders"],
    on_assign=on_assign,
    on_revoke=on_revoke,
)
```

Rebalance callbacks matter when processing state must be flushed or partitions must be paused and resumed safely.

## Python producer with retry-safe delivery

```python
from confluent_kafka import Producer

producer = Producer({
    "bootstrap.servers": "localhost:9092",
    "acks": "all",
    "enable.idempotence": True,
    "retries": 10,
    "delivery.timeout.ms": 120000,
})

for index in range(100):
    producer.produce(
        "events",
        key=f"entity-{index % 10}",
        value=f"event-{index}".encode(),
    )
    producer.poll(0)

producer.flush()
```

## Python transactions

```python
from confluent_kafka import Consumer, Producer

producer = Producer({
    "bootstrap.servers": "localhost:9092",
    "transactional.id": "transformer-instance-1",
    "enable.idempotence": True,
})
producer.init_transactions()

consumer = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "transformer-group",
    "enable.auto.commit": False,
    "isolation.level": "read_committed",
})
consumer.subscribe(["input-events"])

message = consumer.poll(1.0)
if message is not None and not message.error():
    producer.begin_transaction()
    try:
        producer.produce(
            "output-events",
            key=message.key(),
            value=transform(message.value()),
        )
        producer.send_offsets_to_transaction(
            [message]
            , consumer.consumer_group_metadata()
        )
        producer.commit_transaction()
    except Exception:
        producer.abort_transaction()
        raise
```

Syntax may vary slightly by client version. Verify the installed `confluent-kafka` API before copying transaction code into production. Consumers reading transactional output should use `isolation.level=read_committed`.

## Python Schema Registry concept

```python
from confluent_kafka.schema_registry import SchemaRegistryClient

schema_registry = SchemaRegistryClient({
    "url": "http://localhost:8081",
})

schema = schema_registry.get_latest_version("orders-value")
print(schema.version, schema.schema.schema_str)
```

For production serialization, use the matching Avro, JSON Schema, or Protobuf serializer and configure compatibility deliberately.

---

# 6. Message design and event contracts

## Event envelope

Use a consistent envelope around business data:

```json
{
  "eventId": "evt_123",
  "eventType": "order.created",
  "eventVersion": 1,
  "occurredAt": "2026-09-25T10:00:00Z",
  "producer": "orders-service",
  "correlationId": "req_456",
  "traceId": "trace_789",
  "data": {
    "orderId": "ord_42",
    "total": 49.99
  }
}
```

Useful distinctions:

- Event name: fact that something happened, such as `order.created`.
- Command: request to do something, such as `payment.authorize`.
- State snapshot: current representation, such as `user.profile.updated`.
- Notification: event intended mainly for communication.

## Event naming

Choose stable names and document ownership:

```text
<domain>.<entity>.<past-tense-action>
orders.order.created
payments.payment.authorized
```

Avoid changing a meaning while keeping the same event name.

## Key selection

Choose a key based on ordering and distribution:

- `orderId` if order events must be ordered.
- `accountId` if account balance events must be ordered.
- A composite or sharded key if one entity is extremely hot and strict ordering can be relaxed.

## Schema evolution rules

Safe changes often include:

- Adding optional fields with defaults.
- Adding new enum values only when consumers tolerate unknown values.
- Keeping old fields during a migration.
- Deploying consumers before producers for backward-compatible changes.

Risky changes include:

- Changing a field type.
- Reusing a field for another meaning.
- Removing required fields.
- Reusing Protobuf field numbers.
- Changing key semantics.

## Serialization choices

| Format | Strength | Tradeoff |
|---|---|---|
| JSON | Simple and readable | Larger, weaker type enforcement |
| Avro | Compact, schema registry friendly | Requires tooling and schema discipline |
| Protobuf | Compact and strongly typed | Field-number evolution rules |
| Raw bytes | Flexible | Contract becomes application-specific |

---

# 7. Reliability patterns

## Retry strategy

Retries should be:

- Bounded
- Delayed with exponential backoff
- Randomized with jitter
- Limited to transient errors
- Observable
- Idempotent

Do not retry validation errors forever. Do not retry an authorization failure as if it were a network failure.

## Retry topic flow

```text
orders
  -> consumer
  -> temporary failure
  -> orders.retry.1m
  -> orders.retry.10m
  -> orders.dlt after maximum attempts
```

Include original metadata when republishing. A retry record has a new Kafka offset, so preserve the original offset in headers or the envelope.

## Dead letter flow

```text
Main topic -> processing error -> retry policy -> DLT -> alert + manual replay
```

A DLT is not a trash can. Define ownership, retention, alerting, and replay procedures.

## Idempotent consumer table

Example relational table:

```sql
CREATE TABLE processed_events (
  consumer_name TEXT NOT NULL,
  event_id TEXT NOT NULL,
  processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (consumer_name, event_id)
);
```

Processing transaction:

```sql
BEGIN;
INSERT INTO processed_events(consumer_name, event_id)
VALUES ('billing-service', 'evt_123');
-- If duplicate, skip the business operation.
UPDATE accounts SET balance = balance - 10 WHERE id = 'acct_1';
COMMIT;
```

The event marker and business update must be in the same database transaction.

## Outbox publisher pseudocode

```text
BEGIN database transaction
  update order
  insert order.created into outbox
COMMIT

repeat:
  read unpublished outbox rows
  publish event with eventId
  mark row published
```

If publishing succeeds but marking fails, the event can be published again. Consumers must deduplicate.

## Circuit breaker

A circuit breaker stops calls to a failing dependency for a period:

- Closed: requests flow normally.
- Open: requests fail fast.
- Half-open: a few test requests decide whether to recover.

Use it around slow or failing external dependencies, not as a substitute for fixing consumer lag.

## Timeout

Every external call should have a bounded timeout. A consumer that waits forever on a dependency can stop heartbeating and trigger rebalances.

## Poison message handling

Never let one invalid record permanently block a partition unless strict ordering requires it. Route it to a DLT with enough context to diagnose and replay.

## Ordering and retries

Parallel processing can complete events out of order. If order matters:

- Keep one key on one partition.
- Process one key serially.
- Use sequence numbers and reject or buffer gaps.
- Avoid sending a later event to a retry path that can overtake an earlier event.

## Exactly-once boundaries

Ask exactly once: where?

- Kafka input to Kafka output: transactions can help.
- Kafka to PostgreSQL: use a database transaction and idempotency or an outbox.
- Kafka to email: provider idempotency keys and deduplication are needed.
- Kafka to an external payment API: use the provider's idempotency feature.

---

# 8. Security

## TLS

Use TLS for client-broker communication, especially across networks. Validate broker certificates and avoid disabling certificate verification in production.

## SASL authentication

Example conceptual configuration:

```properties
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="app" password="secret";
```

Store credentials in a secret manager, not source control or container images.

## Authorization

Use ACLs to follow least privilege:

- A producer can write only to its required topics.
- A consumer can read only its topics and group.
- Operators have administrative access separately.
- Applications cannot create arbitrary topics in production.

## Sensitive data

Avoid putting passwords, full payment data, or unnecessary personal data in events. Kafka retention and replication create multiple copies of data.

If sensitive data must be streamed:

- Minimize fields.
- Encrypt where appropriate.
- Restrict topic access.
- Define retention and deletion behavior.
- Understand backups and replicas.
- Consider tokenization.

## Multi-tenant Kafka

Separate tenants with topic boundaries, ACLs, quotas, encryption, and clear naming. Avoid relying only on an application field such as `tenantId` for isolation.

---

# 9. Operations and monitoring

## Important broker metrics

- Under-replicated partitions
- Offline partitions
- Active controller count
- Request latency
- Bytes in and out
- Network processor idle percent
- Request queue time
- Disk usage and disk latency
- ISR shrink and expand rate
- Replica fetcher lag
- Log flush and cleanup activity

## Important producer metrics

- Record send rate
- Record error rate
- Retry rate
- Request latency
- Batch size
- Compression ratio
- Buffer exhaustion
- Delivery timeout count

## Important consumer metrics

- Consumer lag
- Poll latency
- Records consumed rate
- Commit latency and failures
- Rebalance count
- Heartbeat failures
- Processing latency
- Retry and DLT rate

## Lag diagnosis

When lag grows, ask:

1. Is producer traffic higher than normal?
2. Is consumer processing slower?
3. Is one partition hot?
4. Is the consumer blocked by a database or API?
5. Are rebalances happening repeatedly?
6. Are records failing and retrying?
7. Are consumers using all available partitions?
8. Is the broker or network unhealthy?

Do not add consumers blindly. If there are fewer partitions than consumers, additional consumers cannot increase parallelism.

## Health checks

A consumer process being alive does not mean it is healthy. Monitor:

- Ability to poll Kafka
- Heartbeat status
- Assignment count
- Last successful processing time
- Last committed offset
- Error and DLT rate
- Dependency health

## Rebalance diagnosis

Frequent rebalances can be caused by:

- Consumer crashes
- Long processing exceeding poll limits
- Network instability
- Coordinator failure
- Deployments
- Too-small session or poll settings

Fix the root cause. Increasing timeouts without measuring processing time can hide failures and increase recovery time.

## Capacity planning

Estimate:

```text
required consumer capacity = peak records per second / records per second per consumer
required partitions >= required active consumers
```

Also consider record size, processing time, database capacity, replication traffic, disk, and network.

## Backup and disaster recovery

Kafka replication is not a complete backup. Plan:

- Cross-cluster replication where required
- Topic configuration backup
- Schema Registry backup
- ACL backup
- Consumer group recovery
- RPO and RTO
- Replay and data validation procedures

Tools such as MirrorMaker 2 can replicate topics between clusters.

## Rolling upgrade

A safe upgrade usually considers:

- Broker protocol compatibility
- Message format compatibility
- Client compatibility
- Controller quorum compatibility
- Rack and replica health
- Consumer rebalance impact

Read the version-specific upgrade documentation before changing production brokers.

---

# 10. Local development example

A simple local Kafka deployment can use the official Kafka image in KRaft mode. Docker image environment names vary by image version, so follow the image's current documentation.

Example conceptual Docker Compose configuration:

```yaml
services:
  kafka:
    image: apache/kafka:latest
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
```

For production, do not copy single-node development values such as replication factor 1.

Create a topic with Kafka CLI:

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic orders \
  --partitions 3 \
  --replication-factor 1
```

Describe it:

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic orders
```

Produce text records:

```bash
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic orders
```

Consume from the beginning:

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --from-beginning \
  --group demo-consumer
```

Inspect a consumer group:

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group demo-consumer
```

Delete a topic only when the cluster policy permits it:

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --delete \
  --topic orders
```

---

# 11. Common mistakes and fixes

## Mistake: assuming Kafka is a queue that deletes messages after reading

Kafka retains records according to policy. Consumer offsets track progress; they do not delete records.

## Mistake: expecting order across partitions

Only one partition provides total topic order. Use a stable key for per-entity order.

## Mistake: using too many consumers

A group cannot actively process more partitions than the topic has.

## Mistake: committing before processing

This can lose records on a crash. Commit after successful processing when using at-least-once delivery.

## Mistake: not making consumers idempotent

At-least-once delivery creates duplicates during crashes and retries. Deduplicate with event IDs or business keys.

## Mistake: infinite retries

A poison pill can block a partition forever. Use bounded retries and a DLT.

## Mistake: publishing database events with two unrelated writes

Use an outbox or CDC to avoid the dual-write problem.

## Mistake: using one partition for massive traffic

One partition limits consumer parallelism. Increase partitions while considering ordering and key distribution.

## Mistake: adding partitions without considering keys

Changing partition count can change key mapping and break assumptions about ordering or state locality.

## Mistake: putting large data in Kafka messages

Use object storage for large files and publish a reference event. Configure message size limits deliberately.

## Mistake: auto-creating topics in production

A typo in a topic name can silently create a new topic. Create topics through reviewed configuration.

## Mistake: ignoring advertised listeners

Clients may connect to the bootstrap broker but fail when Kafka returns an unreachable advertised address.

## Mistake: using `latest` casually

A new consumer group with `latest` skips existing records. Choose `earliest` or `latest` intentionally.

## Mistake: treating replication as backup

Replication protects broker failure but does not protect against accidental deletion, corruption, or bad producers.

## Mistake: logging secrets in events

Events are retained, replicated, backed up, and consumed by many systems. Minimize sensitive data.

---

# 12. Testing Kafka applications

## Unit tests

Test event mapping and business logic without Kafka:

```python
def build_order_event(order):
    return {
        "type": "order.created",
        "orderId": order["id"],
    }


def test_build_order_event():
    assert build_order_event({"id": "42"}) == {
        "type": "order.created",
        "orderId": "42",
    }
```

## Integration tests

Run a real Kafka broker in a test container or ephemeral environment. Test:

- Producing and consuming
- Consumer group behavior
- Partition keys
- Offset commits
- Rebalances
- Retry topics
- DLT behavior
- Schema compatibility
- Transactional output where applicable

## Contract tests

Verify that producers publish fields consumers require and that schemas follow compatibility rules.

## Failure tests

Simulate:

- Broker unavailable
- Producer request timeout
- Consumer restart
- Duplicate delivery
- Malformed payload
- Slow database
- Rebalance during processing
- Full disk or unavailable dependency

## Testcontainers concept

Use a disposable Kafka container in CI where practical. Keep tests isolated with unique topic names and consumer group IDs.

## Replay testing

A valuable Kafka test is replaying retained records into a new group and verifying the resulting state. Make replay safe through idempotency and deterministic processing.

---

# 13. Kafka interview questions and answers

## Fundamentals

### What problem does Kafka solve?

Kafka provides durable, distributed event storage and high-throughput pub/sub. Producers and consumers are decoupled in time and scale, and multiple consumer groups can independently read the same events.

### Kafka versus a traditional message queue?

A queue often focuses on handing work to one consumer and removing or acknowledging messages. Kafka retains an ordered log, allows replay by offset, partitions for scale, and lets multiple consumer groups read independently. Modern queues can also have durable streams, so compare the actual requirements rather than relying on labels.

### What is a topic?

A named logical stream of records divided into partitions.

### What is a partition?

An ordered append-only log and the unit of Kafka parallelism. Ordering is guaranteed only inside a partition.

### What is an offset?

A sequential position within one partition. Consumers use offsets to track progress.

### What is a broker?

A Kafka server that stores partition replicas and serves producer and consumer requests.

### What is a consumer group?

A set of consumers that share work. Each partition is assigned to at most one active consumer in a group.

### Can two consumer groups read the same topic?

Yes. Each group has independent offsets and receives the topic's records independently.

### What happens if there are more consumers than partitions?

Some consumers are idle because a partition can be assigned to only one consumer in a group.

### How is a record assigned to a partition?

Usually the key is hashed to select a partition. Without a key, the producer's partitioner selects partitions according to client behavior.

## Delivery and ordering

### What does at-least-once mean?

A record is processed one or more times. A crash after processing but before committing the offset can cause a duplicate.

### How do you implement at-most-once?

Commit the offset before processing. This reduces duplicates but can lose records if processing fails afterward.

### How do you handle duplicates?

Use an event ID or business key with a unique database constraint, and perform deduplication and the business update in one transaction.

### How do you guarantee order?

Use the same key for events that must be ordered so they go to the same partition, then process that partition or key serially. Kafka does not provide global order across partitions.

### What is `acks=all`?

The producer waits for acknowledgement from all required in-sync replicas. It provides stronger durability than `acks=0` or `acks=1`, with more latency.

### What is idempotent producer mode?

It prevents duplicate records caused by producer retries within the producer session. It does not make arbitrary external side effects idempotent.

### What are Kafka transactions?

Transactions can atomically write multiple Kafka records and commit consumed offsets, enabling exactly-once processing between Kafka input and Kafka output when configured correctly.

### Does Kafka guarantee exactly-once delivery to a database?

No. Kafka transactions do not automatically include an external database. Use an outbox, database transaction, idempotency, or the external system's idempotency feature.

## Consumer groups and operations

### What causes a rebalance?

Consumers joining or leaving, crashes, partition changes, subscription changes, or missed heartbeats and poll deadlines.

### What is consumer lag?

The distance between the end of available records and a consumer's position or committed position. Sustained growing lag means consumption is not keeping up.

### How do you reduce consumer lag?

Find the bottleneck first. Then add partitions and consumers if partition capacity allows, optimize processing, batch I/O, scale dependencies, reduce retries, and fix hot partitions.

### What does `auto.offset.reset` do?

It selects the starting point when no valid committed offset exists. `earliest` starts with retained records, `latest` starts with new records, and `none` fails.

### Why might a consumer receive duplicate records after restart?

Because processing may have succeeded but the offset commit may not have. This is expected under at-least-once processing.

### What is a poison pill?

A record that repeatedly fails processing. Route it to a retry topic or DLT instead of retrying forever.

## Architecture

### How many partitions should a topic have?

Enough for expected throughput and consumer parallelism, with growth capacity. Consider key distribution, ordering, broker resources, and the cost of future partition changes.

### What is replication factor?

The number of copies of each partition across brokers.

### What are ISR members?

Replicas sufficiently caught up with the leader to be eligible for leadership under normal safe election rules.

### Why use `min.insync.replicas` with `acks=all`?

To require a minimum number of healthy replicas before acknowledging writes, improving durability during broker failures.

### What is log compaction?

A cleanup policy that retains the latest value for each key, useful for reconstructing current state. It is not the same as ordinary time-based retention.

### Kafka versus database?

Kafka is optimized for durable distributed event streams and sequential append/read patterns. A database provides indexed queries, constraints, transactions, and current-state access. Many systems use both.

### Kafka versus Redis Pub/Sub?

Kafka retains records and supports replay and consumer groups. Redis Pub/Sub is primarily transient delivery; subscribers that are offline generally miss messages. Redis Streams provide different durable stream capabilities and tradeoffs.

### What is the outbox pattern?

Write the business change and an event row in one database transaction, then publish the outbox row asynchronously. It avoids inconsistent dual writes, with duplicates handled by idempotent consumers.

### What is CDC?

Change Data Capture publishes database changes as events, often using Kafka Connect and Debezium.

## Security and schema

### How do you secure Kafka?

Use TLS for encryption and broker/client authentication, SASL for authentication where appropriate, ACLs for authorization, quotas for noisy clients, secret management, least privilege, and data minimization.

### What is Schema Registry?

A service that stores and versions event schemas and can enforce compatibility rules for Avro, Protobuf, or JSON Schema.

### Backward versus forward compatibility?

Backward compatibility lets new consumers read old data. Forward compatibility lets old consumers read new data. Full compatibility supports both directions.

### Why should event schemas be versioned?

Events are long-lived contracts consumed by independently deployed applications. Explicit schema evolution prevents silent breakage.

## Node.js and Python

### How do you produce with KafkaJS?

Create a Kafka client, connect a producer, call `send()` with a topic and messages, handle delivery errors, and disconnect on shutdown. Use stable keys and idempotent configuration where appropriate.

### How do you consume with KafkaJS?

Create a consumer with a `groupId`, connect, subscribe to topics, and call `run()` with `eachMessage` or `eachBatch`. Commit offsets only according to the processing guarantee required.

### How do you consume with `confluent-kafka`?

Create a `Consumer`, subscribe, repeatedly call `poll()`, process successful messages, commit after processing, and call `close()` on shutdown.

### Why must Python call `poll()`?

The consumer poll loop fetches records and allows the client to handle group coordination and callbacks. A consumer that does not poll can lose membership or fail to progress.

### How do you handle long-running work in a consumer?

Use a worker pool or queue, pause partitions when needed, keep heartbeats and polling healthy, commit only completed work, and configure timeouts based on measured processing time.

---

# 14. Practical design exercise: order processing

## Requirement

When an order is created:

1. Save the order.
2. Publish `order.created`.
3. Create an invoice.
4. Send a notification.
5. Allow analytics to consume the event independently.

## Design

```text
Orders API
   |
   | database transaction
   v
Orders table + outbox table
                 |
                 v
            Outbox publisher
                 |
                 v
             orders topic
          /       |        \
         v        v         v
    Billing    Notify    Analytics
```

Use `orderId` as the Kafka key so order events stay ordered.

## Failure handling

- Database succeeds but process crashes before publishing: outbox publisher retries.
- Kafka publish succeeds but outbox update fails: duplicate publish is possible; consumers deduplicate using `eventId`.
- Billing fails temporarily: retry topic with backoff.
- Billing fails permanently: DLT and alert.
- Notification provider is down: queue notification work separately.
- Analytics is down: its consumer group falls behind while orders remain available.

## Interview tradeoffs

- More partitions increase throughput but complicate ordering and operations.
- Longer retention supports replay but costs storage.
- Synchronous publishing lowers uncertainty for the caller but increases request latency.
- Asynchronous publishing improves API latency but requires status and failure handling.
- Exactly-once Kafka processing does not guarantee exactly-once email or payment effects.

---

# 15. Production checklist

## Design

- [ ] Topic ownership is documented.
- [ ] Event names, keys, schemas, and versions are documented.
- [ ] Ordering requirements are explicit.
- [ ] Partition count is based on throughput and parallelism.
- [ ] Retention and compaction policies are intentional.
- [ ] Replay and deletion behavior are understood.

## Producers

- [ ] `acks` matches durability requirements.
- [ ] Idempotence is enabled where appropriate.
- [ ] Keys are stable and distribute traffic.
- [ ] Retries are bounded and observable.
- [ ] Payload size is limited.
- [ ] Delivery errors are handled.

## Consumers

- [ ] Consumer groups have clear ownership.
- [ ] Offset commit timing is deliberate.
- [ ] Handlers are idempotent.
- [ ] Long processing cannot silently lose group membership.
- [ ] Rebalances are tested.
- [ ] Lag is monitored.
- [ ] Retry and DLT behavior is defined.
- [ ] Shutdown drains or stops safely.

## Platform

- [ ] Multiple brokers are used for production.
- [ ] Replication factor and `min.insync.replicas` are appropriate.
- [ ] Rack or availability-zone awareness is configured.
- [ ] TLS, authentication, and ACLs are enabled.
- [ ] Advertised listeners are reachable by all clients.
- [ ] Disk, network, and controller metrics are monitored.
- [ ] Topic creation is controlled.
- [ ] Backups or cross-cluster recovery are tested.

## Application

- [ ] Events do not contain unnecessary secrets or personal data.
- [ ] Schema compatibility is enforced.
- [ ] Database dual writes use an outbox or CDC.
- [ ] External side effects use idempotency keys.
- [ ] Timeouts and circuit breaking are defined.
- [ ] Integration and failure tests exist.
- [ ] Runbooks explain replay, DLT, lag, and rollback procedures.

---

# 16. Quick revision sheet

```text
Topic       = named stream
Partition   = ordered log and unit of parallelism
Offset      = position within a partition
Broker      = Kafka server
Leader      = replica serving normal reads and writes
Follower    = replica copying the leader
ISR         = replicas caught up enough for safe leadership
Producer    = writes records
Consumer    = reads records
Group       = consumers sharing partitions
Lag         = records behind the log end
Commit      = saved next position for a group
Retention   = how long or how much data is kept
Compaction  = keep latest value for each key
DLT         = failed records isolated for diagnosis/replay
Schema      = event contract
Connect     = integration framework
CDC         = database changes as events
KRaft       = Kafka metadata consensus mode
Outbox      = reliable database-to-event publishing pattern
```

The best Kafka interview answers explain both the mechanism and the tradeoff: ordering versus throughput, durability versus latency, retries versus duplicates, retention versus cost, and availability versus consistency.
