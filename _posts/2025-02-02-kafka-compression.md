---
title: "Kafka Message compression"
layout: single
date: 2025-02-02
permalink: /posts/kafka-message-compression/
tags:
  - Kafka
---
This article explains how Kafka message compression works, its configuration, and considerations for both producers and consumers.

{% include toc %}

## Introduction

- Kafka producer data compression works by `batching data going to the same partition` before applying compression.

  - Batching: Messages going to the same partition are grouped together before compression.
  - Compression Type: The chosen compression algorithm (e.g., Snappy, Gzip) affects performance.
  - Decompression: Brokers need to decompress some batches for validation or compaction purposes.

- Why the same partition, and not at the topic level?

  - If you compress data for multiple partitions in a single batch, then that batch would have to go to multiple leaders, which would send more data over the wire, making compression not worth the effort.

- Batch size trade off:
  - Small Batch Size: Saves memory, reduces latency (good for low-throughput, low-latency scenarios).
  - Large Batch Size: Increases throughput, but consumes more memory (good for high-throughput scenarios).

## Compression Types

![Compression Types]({{ site.baseurl }}/images/blogs/compression.png)

## Configuration

- **Producer Configuration:**
  - `compression.type`: The compression algorithm to use.
  - `batch.size`: The maximum size of a batch before compression.
  - `linger.ms`: The time to wait for more messages before sending a batch.
  - `max.request.size`: The maximum size of a request.

## Kafka consumers

- `Compatibility`: Consumers can handle both compressed and uncompressed messages, allowing flexibility in producer-consumer interaction.
- `Consumer Handling`: Consumers recognize compressed messages via a special header and decompress them before returning decompressed messages.
- `Decoupling Advantage`: Kafka's design allows decoupling between producers and consumers, facilitating the handling of mixed message types.

## Notes

- Encrypted data should not be compressed since data is random.

## Reference

- <https://www.confluent.io/blog/apache-kafka-message-compression/>
