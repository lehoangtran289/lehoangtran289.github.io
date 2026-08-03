---
title: "Kafka Best Practices"
layout: single
date: 2025-02-01
permalink: /posts/kafka-best-practices/
author_profile: false
tags:
  - Kafka
  - Best Practices
---
Below are some notes on Kafka best practices that I found useful while working with Kafka in production.

{% include toc %}

## 1. Choosing topic/Partitions

- Topic/Partition is unit of parallelism in Kafka
- More Partitions = more parallelism of consumers -> More throughput

### Consideration

- More partitions can increase the end-to-end latency (time from producer to consumer read)

  - Messages are exposed to consumers only after being committed (replicated to all in-sync replicas)
  - Replicating 1000 partitions can take 20ms, potentially too high for real-time applications.

- Kafka Producer Buffering mechanism:

  - Messages are buffered per partition and sent to the broker after enough accumulation or time.
  - more partitions = messages will be accumulated for more partitions on producer side.
    - Each partition has its own buffer, and messages are stored in batches within these buffers
    - More partitions = producer needs to maintain more buffers

- Kafka Consumer:
  - fetches batch of messages per partitions. The more partitions that consumer is subscribing to, the more memory it needs.

## 2. Factors Affecting Performance

- RAM - Main memory. More specifically File system buffer cache.
- Multiple dedicated disks.
- **Partitions per topic**. More partitions allows increased parallelism.
- Ethernet bandwidth.

## 3. Kafka Broker configs

- `log.retention.hours`: Controls when the old messages in a topic will be deleted.
- `message.max.bytes`: Maximum message size the server can accept. Ensure `replica.fetch.max.bytes` is set to be >= this value.
- `delete.topic.enable`: Allows users to delete a topic from Kafka. Disabled by default, functional from Kafka 0.9 onwards.
- `unclean.leader.election`: true by default, prioritizing availability over durability.
  - If set to true, a replica can become the leader without it being a ISR if the original leader broker goes down, risking data loss.
  - Set to false for higher durability to prevent unclean leader elections.

## 4. Kafka producer

### Critical configs

- `batch.size` (size based batching)
- `linger.ms` (time based batching)
- `compression.type`
- `max.in.flight.requests.per.connection` (affects ordering)
- `acks` (affects durability)

### Performance notes

- **Partition Targeting:**

  - A producer thread targeting a single partition performs faster than one sending messages to multiple partitions.

- **Flush Optimization:**

  - The new Producer API's flush() call can improve performance if used correctly.
  - Optimal performance is achieved with around 4MB of data between flush() calls (based on 1KB event size).
  - Rule of thumb for batch size: `batch.size = total bytes between flush() / partition count.`

- **Scaling Producers:**

  - If producer throughput is maximized but there is spare CPU and network capacity, add more producer processes.

- **Event Size Sensitivity:**

  - Larger event sizes (e.g., 1KB) generally yield better throughput than smaller events (e.g., 100 bytes).

- **Linger.ms Impact:**
  - No definitive rule; its impact varies with use case. For small events (100 bytes or less), it generally has minimal effect.

### Lifecycle of a request from Producer to Broker

- **Batch Polling:** Polls a batch from the batch queue, one batch per partition.

- **Batch Grouping:** Groups batches based on the leader broker then sends the grouped batches to the brokers.

- **Pipelining**: If `max.in.flight.requests.per.connection` > 1, requests are pipelined.

- **Batch Readiness:** A batch is ready to send when:

  - `batch.size` is reached.
  - `linger.ms` is reached.
  - Another batch to the same broker is ready.
  - flush() or close() is called.

- **Big Batching:**
  - Results in better compression ratio and higher throughput.
  - Leads to higher latency.

### Compression.type

- Compression is in user thread, so adding more threads helps with the throughput if compression is slow

![Compression Types]({{ site.baseurl }}/images/blogs/compression.png)

### ACKs

- Defines durability level for producer.

### Max.in.flight.requests.per.connection

- Max.in.flight.requests.per.connection > 1 means pipelining.
  - Gives better throughput
  - May cause out of order delivery when retry occurs
  - Excessive pipelining , drops throughput

## 5. Kafka consumer

- Rule of thumb:
  - Number of consumer threads = Partition count
  - Microbenchmarking showed that `Consumer performance was not as sensitive to event size or batch size as compared to Producer`. Both 1kb and 100byte events showed similar throughput.

## References

- <https://community.cloudera.com/t5/Community-Articles/Kafka-Best-Practices/ta-p/249371>
- <https://www.redpanda.com/guides/kafka-performance-kafka-rebalancing>
- <https://dev.to/konstantinas_mamonas/kafka-producers-explained-partitioning-batching-and-reliability-4bm8>
- <https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/?placement=&device=c&creative=&session_ref=https%3A%2F%2Fwww.google.com%2F)>