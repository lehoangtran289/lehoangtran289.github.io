---
title: "Spring Kafka Error Handling, with Best Practices"
layout: single
date: 2026-02-14
permalink: /posts/spring-kafka-error-handling/
author_profile: false
tags:
  - Kafka
  - Spring Boot
  - Java
  - Best Practices
---

This article covers error handling strategies in Spring Kafka, including offset management, retry mechanisms, dead letter topics, batch error handling, and common pitfalls to avoid when building resilient Kafka consumers.

Note that: This article is AI-generated based on the latest Spring Kafka documentation and best practices from several references.

{% include toc %}

## 1. Understanding Error Handlers

### 1.1. The Basics

Spring Kafka provides container-level error handlers that catch exceptions thrown by your `@KafkaListener` methods. These handlers determine what happens when message processing fails.

Starting with version 2.8, Spring Kafka introduced `CommonErrorHandler` to replace the legacy `ErrorHandler` and `BatchErrorHandler` interfaces. This unified approach handles both record and batch listeners with a single configuration.

**Key points:**

- The error handler runs after your listener throws an exception
- By default, `DefaultErrorHandler` is used with 10 retry attempts
- Error handlers can seek back to failed offsets or skip records entirely
- Works for both transactional and non-transactional listeners

### 1.2. DefaultErrorHandler Overview

The `DefaultErrorHandler` replaced `SeekToCurrentErrorHandler` and `RecoveringBatchErrorHandler`. It provides retry logic with configurable backoff strategies.

**Default behavior:**

- 10 retry attempts with no delay between attempts
- After retries exhausted → logs the failed record at ERROR level
- By default, seeks back to the failed offset for redelivery
- Can be configured with a custom recoverer (like publishing to DLT)

**Basic configuration example:**

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, Order> kafkaListenerContainerFactory(
        ConsumerFactory<String, Order> consumerFactory) {
    
    ConcurrentKafkaListenerContainerFactory<String, Order> factory = 
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory);
    
    // Configure error handler with 3 retry attempts, 1 second between retries
    DefaultErrorHandler errorHandler = new DefaultErrorHandler(
        new FixedBackOff(1000L, 2L)  // 1s delay, 2 retries = 3 total attempts
    );
    
    factory.setCommonErrorHandler(errorHandler);
    return factory;
}
```

**Fatal exceptions** — these skip retries entirely:

- `DeserializationException`
- `MessageConversionException`
- `ConversionException`
- `MethodArgumentResolutionException`
- `NoSuchMethodException`
- `ClassCastException`

These are considered fatal because retrying won't fix the issue. The recoverer is invoked immediately.

## 2. Offset Commit Strategies

### 2.1. Understanding AckMode

Spring Kafka provides different acknowledgment modes that control when offsets are committed. This is critical for understanding message delivery guarantees.

**Important:** By default, Spring Kafka sets `enable.auto.commit=false`. The framework manages offset commits, not Kafka itself.

**Available AckMode values:**

- `BATCH` (default) — Commits offset when all records from the poll are processed
- `RECORD` — Commits offset after each record is processed
- `MANUAL` — Application calls `Acknowledgment.acknowledge()`, commit happens when the whole batch completes
- `MANUAL_IMMEDIATE` — Application calls `Acknowledgment.acknowledge()`, commit happens immediately

### 2.2. BATCH Mode (Default)

With `BATCH` mode, Spring Kafka processes messages one by one but only commits the offset after the entire batch from the poll completes.

**Use case:**

- Best for throughput
- Acceptable to reprocess some messages after failures
- Handles most standard scenarios

**Properties:**

```yaml
spring:
  kafka:
    listener:
      ack-mode: BATCH  # This is the default
```

**Behavior:**

```
Poll returns: [msg1, msg2, msg3, msg4, msg5]
Process msg1 → success
Process msg2 → success
Process msg3 → FAILURE
```

What happens:

1. Container commits offsets for msg1 and msg2
2. Error handler seeks back to offset for msg3
3. Next poll returns [msg3, msg4, msg5] again
4. If app crashes before commit → msg1 and msg2 will be reprocessed

**Trade-off:** Higher throughput, but potential for duplicate processing if the consumer crashes between processing and commit.

### 2.3. RECORD Mode

With `RECORD` mode, the offset is committed immediately after each message is processed successfully.

**Use case:**

- Minimize duplicate processing after restarts
- Each message is expensive to process
- You can tolerate lower throughput

**Properties:**

```yaml
spring:
  kafka:
    listener:
      ack-mode: RECORD
```

**Behavior:**

```
Process msg1 → commit offset for msg1
Process msg2 → commit offset for msg2
Process msg3 → FAILURE → error handling
```

**Trade-off:** Lower throughput (more commits), but fewer duplicates after failures.

### 2.4. MANUAL_IMMEDIATE Mode

This gives you full control. You decide exactly when to commit offsets by calling `Acknowledgment.acknowledge()`.

**Use case:**

- Processing messages asynchronously with custom thread pools
- Need precise control over commit timing
- Building exactly-once semantics with external state

**Properties:**

```yaml
spring:
  kafka:
    listener:
      ack-mode: MANUAL_IMMEDIATE
```

**Example code:**

```java
@KafkaListener(topics = "orders", groupId = "order-processor")
public void listen(@Payload Order order,
                   Acknowledgment acknowledgment) {
    try {
        processOrder(order);
        // Only commit if processing succeeds
        acknowledgment.acknowledge();
    } catch (Exception e) {
        log.error("Failed to process order: {}", order, e);
        // Don't acknowledge → message will be redelivered
    }
}
```

**Async processing example:**

```java
@Service
public class AsyncListener {

    private final ExecutorService executorService = 
        Executors.newFixedThreadPool(20);
    
    @Autowired
    private OrderProcessor processor;
    
    @KafkaListener(topics = "orders")
    public void listenAsync(@Payload Order order,
                           Acknowledgment acknowledgment) {
        // Submit to thread pool
        executorService.submit(() -> {
            try {
                processor.process(order);
                // Acknowledge after async processing completes
                acknowledgment.acknowledge();
            } catch (Exception e) {
                log.error("Async processing failed", e);
                // Don't acknowledge
            }
        });
    }
}
```

### 2.5. Async Acknowledgments (async-acks)

By default, out-of-order acknowledgments are disabled. If you're using `MANUAL` or `MANUAL_IMMEDIATE` with async processing, you might want to enable this.

**Properties:**

```yaml
spring:
  kafka:
    listener:
      ack-mode: MANUAL_IMMEDIATE
      async-acks: true  # Allow out-of-order commits
```

**When to use:**

- Processing messages with a custom thread pool
- Messages can complete in different order than received
- High-throughput scenarios where ordering isn't critical

**Warning:** Enabling `async-acks` means offsets might be committed out of order. If message 5 finishes before message 3, offset 5 gets committed first. If the app crashes, message 3 and 4 might not be redelivered.

## 3. Dead Letter Topics (DLT)

### 3.1. Publishing Failed Records to DLT

After retries are exhausted, you typically want to send failed messages somewhere for later analysis rather than just logging them.

`DeadLetterPublishingRecoverer` publishes failed records to a dead letter topic.

**Basic setup:**

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, Order> kafkaListenerContainerFactory(
        ConsumerFactory<String, Order> consumerFactory,
        KafkaTemplate<String, Order> kafkaTemplate) {
    
    ConcurrentKafkaListenerContainerFactory<String, Order> factory = 
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory);
    
    // Create recoverer that sends to DLT
    DeadLetterPublishingRecoverer recoverer = 
        new DeadLetterPublishingRecoverer(kafkaTemplate);
    
    // Create error handler with recoverer
    DefaultErrorHandler errorHandler = new DefaultErrorHandler(
        recoverer,
        new FixedBackOff(1000L, 2L)  // 3 attempts: initial + 2 retries
    );
    
    factory.setCommonErrorHandler(errorHandler);
    return factory;
}
```

**Default behavior:**

- Failed records are sent to `<originalTopic>.DLT`
- Same partition as original record
- DLT topic must have at least as many partitions as original topic

### 3.2. Custom DLT Routing

You can customize which DLT to use based on the exception type or record content.

**Example:**

```java
DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
    kafkaTemplate,
    (record, exception) -> {
        if (exception.getCause() instanceof ValidationException) {
            return new TopicPartition(record.topic() + ".validation.DLT", record.partition());
        } else if (exception.getCause() instanceof DatabaseException) {
            return new TopicPartition(record.topic() + ".database.DLT", record.partition());
        } else {
            return new TopicPartition(record.topic() + ".DLT", record.partition());
        }
    }
);
```

### 3.3. DLT Headers

The recoverer adds several headers to the DLT record for debugging:

- `kafka_dlt-exception-fqcn` — Exception class name
- `kafka_dlt-exception-cause-fqcn` — Root cause class name
- `kafka_dlt-exception-message` — Exception message
- `kafka_dlt-exception-stacktrace` — Full stack trace
- `kafka_dlt-original-topic` — Original topic name
- `kafka_dlt-original-partition` — Original partition
- `kafka_dlt-original-offset` — Original offset
- `kafka_dlt-original-timestamp` — Original timestamp
- `kafka_dlt-original-consumer-group` — Consumer group

**These headers help you:**

- Debug what went wrong
- Know where the message came from
- Implement retry logic from the DLT later

### 3.4. Skipping Retries for Specific Exceptions

The `DefaultErrorHandler` considers certain exceptions fatal by default. You can add your own.

**Example:**

```java
@Bean
public DefaultErrorHandler errorHandler(
        KafkaTemplate<String, Order> kafkaTemplate) {
    
    DeadLetterPublishingRecoverer recoverer = 
        new DeadLetterPublishingRecoverer(kafkaTemplate);
    
    DefaultErrorHandler handler = new DefaultErrorHandler(
        recoverer,
        new FixedBackOff(1000L, 2L)
    );
    
    // Add custom non-retryable exceptions
    handler.addNotRetryableExceptions(
        IllegalArgumentException.class,
        ValidationException.class,
        BadRequestException.class
    );
    
    return handler;
}
```

Now these exceptions skip retries and go straight to the DLT on first failure.

## 4. Batch Listener Error Handling

### 4.1. Using BatchListenerFailedException

For batch listeners, you must throw `BatchListenerFailedException` to indicate which specific record in the batch failed.

**Basic batch listener:**

```java
@KafkaListener(
    id = "order-batch-processor",
    topics = "orders",
    containerFactory = "batchFactory"
)
public void processBatch(List<ConsumerRecord<String, Order>> records) {
    for (ConsumerRecord<String, Order> record : records) {
        try {
            processOrder(record.value());
        } catch (Exception e) {
            // Identify the failed record
            throw new BatchListenerFailedException("Failed to process", e, record);
        }
    }
}
```

**For POJO batch listeners** (no `ConsumerRecord`):

```java
@KafkaListener(topics = "orders", containerFactory = "batchFactory")
public void processBatch(List<Order> orders) {
    for (int i = 0; i < orders.size(); i++) {
        try {
            processOrder(orders.get(i));
        } catch (Exception e) {
            // Use index instead of record
            throw new BatchListenerFailedException("Failed to process", e, i);
        }
    }
}
```

### 4.2. Batch Error Handler Configuration

Configure the batch factory with error handling:

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, Order> batchFactory(
        ConsumerFactory<String, Order> consumerFactory,
        KafkaTemplate<String, Order> kafkaTemplate) {
    
    ConcurrentKafkaListenerContainerFactory<String, Order> factory = 
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory);
    factory.setBatchListener(true);  // Enable batch mode
    
    DeadLetterPublishingRecoverer recoverer = 
        new DeadLetterPublishingRecoverer(kafkaTemplate);
    
    DefaultErrorHandler errorHandler = new DefaultErrorHandler(
        recoverer,
        new FixedBackOff(1000L, 2L)
    );
    
    factory.setCommonErrorHandler(errorHandler);
    return factory;
}
```

### 4.3. How Batch Error Handling Works

When a record in a batch fails:

1. Offsets for records before the failed record are committed
2. Failed record (and remaining records) are retried according to BackOff
3. After retries exhausted → only the failed record goes to DLT
4. Failed record's offset is committed
5. Remaining records are redelivered for processing

**Example flow:**

```
Batch: [msg1, msg2, msg3, msg4, msg5]
Process msg1 → success
Process msg2 → success
Process msg3 → FAILURE
```

What happens:

1. Commit offsets for msg1, msg2
2. Retry attempt 1: [msg3, msg4, msg5] → still fails on msg3
3. Retry attempt 2: [msg3, msg4, msg5] → still fails on msg3
4. Retry attempt 3: [msg3, msg4, msg5] → still fails on msg3
5. Send msg3 to DLT, commit offset for msg3
6. Continue with [msg4, msg5]

### 4.4. Batch Processing with Thread Pools

Processing batch messages asynchronously with a thread pool is a common pattern for maximizing throughput. However, it requires careful error handling to ensure no message loss.

#### 4.4.1. The Challenge

When you submit batch records to a thread pool, the `@KafkaListener` method returns immediately. If you're not careful, offsets get committed before async processing completes → messages can be lost if processing fails.

**Basic setup (naive approach):**

```java
@Configuration
public class BatchAsyncConfig {
    
    @Bean
    public ExecutorService kafkaProcessorPool() {
        return new ThreadPoolExecutor(
            10,  // core threads
            50,  // max threads
            60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(1000)
        );
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Order> batchFactory(
            ConsumerFactory<String, Order> consumerFactory) {
        
        ConcurrentKafkaListenerContainerFactory<String, Order> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setBatchListener(true);
        
        // CRITICAL: Use MANUAL_IMMEDIATE for async processing
        factory.getContainerProperties().setAckMode(
            ContainerProperties.AckMode.MANUAL_IMMEDIATE
        );
        
        return factory;
    }
}
```

#### 4.4.2. Reliable Async Batch Processing

Use `CompletableFuture` to track all async tasks and only acknowledge after all complete successfully.

**Complete implementation:**

```java
@Service
public class AsyncBatchListener {
    
    private static final Logger log = LoggerFactory.getLogger(AsyncBatchListener.class);
    
    @Autowired
    private ExecutorService kafkaProcessorPool;
    
    @Autowired
    private OrderProcessor orderProcessor;
    
    @KafkaListener(
        topics = "orders",
        containerFactory = "batchAsyncFactory"
    )
    public void processBatch(
            List<ConsumerRecord<String, Order>> records,
            Acknowledgment acknowledgment) {
        
        log.info("Received batch of {} records", records.size());
        
        // Create a future for each record
        List<CompletableFuture<Void>> futures = new ArrayList<>();
        
        for (int i = 0; i < records.size(); i++) {
            final int index = i;
            final ConsumerRecord<String, Order> record = records.get(i);
            
            CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
                try {
                    orderProcessor.process(record.value());
                    log.debug("Successfully processed record at index {}", index);
                } catch (Exception e) {
                    log.error("Failed to process record at index {}", index, e);
                    throw new CompletionException(e);
                }
            }, kafkaProcessorPool);
            
            futures.add(future);
        }
        
        // Wait for all futures to complete
        try {
            CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
                .join();  // Blocks until all complete
            
            // All succeeded → commit offset
            acknowledgment.acknowledge();
            log.info("Batch processed successfully, offset committed");
            
        } catch (CompletionException e) {
            log.error("Batch processing failed, offset NOT committed", e);
            // Don't acknowledge → entire batch will be redelivered
            // This ensures no message loss but can cause duplicates
        }
    }
}
```

**Key points:**

- Uses `MANUAL_IMMEDIATE` ack mode
- `CompletableFuture.allOf()` waits for all tasks to complete
- Only calls `acknowledge()` if all tasks succeed
- If any task fails → batch is redelivered

**Trade-off:** This approach guarantees no message loss but means the entire batch is reprocessed if any single message fails.

#### 4.4.3. Per-Record Acknowledgment with Async Processing

For finer-grained control, acknowledge successful records individually while handling failures separately.

**Advanced implementation:**

```java
@Service
public class FinegrainedAsyncBatchListener {
    
    private static final Logger log = LoggerFactory.getLogger(FinegrainedAsyncBatchListener.class);
    
    @Autowired
    private ExecutorService kafkaProcessorPool;
    
    @Autowired
    private OrderProcessor orderProcessor;
    
    @Autowired
    private KafkaTemplate<String, Order> kafkaTemplate;
    
    @KafkaListener(
        topics = "orders",
        containerFactory = "batchAsyncFactory"
    )
    public void processBatch(
            List<ConsumerRecord<String, Order>> records,
            Acknowledgment acknowledgment) {
        
        log.info("Processing batch of {} records", records.size());
        
        // Track failed records
        ConcurrentHashMap<Integer, ConsumerRecord<String, Order>> failedRecords = 
            new ConcurrentHashMap<>();
        
        CountDownLatch latch = new CountDownLatch(records.size());
        
        for (int i = 0; i < records.size(); i++) {
            final int index = i;
            final ConsumerRecord<String, Order> record = records.get(i);
            
            kafkaProcessorPool.submit(() -> {
                try {
                    orderProcessor.process(record.value());
                    log.debug("Successfully processed record at index {}", index);
                } catch (Exception e) {
                    log.error("Failed to process record at index {}: {}", 
                        index, e.getMessage());
                    failedRecords.put(index, record);
                } finally {
                    latch.countDown();
                }
            });
        }
        
        // Wait for all tasks to complete
        try {
            if (!latch.await(60, TimeUnit.SECONDS)) {
                log.error("Batch processing timed out after 60 seconds");
                // Don't acknowledge → entire batch redelivered
                return;
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("Batch processing interrupted", e);
            return;
        }
        
        // Handle failures
        if (!failedRecords.isEmpty()) {
            log.warn("Batch had {} failures", failedRecords.size());
            
            // Send failed records to DLT
            failedRecords.values().forEach(record -> {
                try {
                    kafkaTemplate.send(
                        record.topic() + ".DLT",
                        record.key(),
                        record.value()
                    );
                } catch (Exception e) {
                    log.error("Failed to send to DLT: {}", record, e);
                }
            });
        }
        
        // Acknowledge the batch (successful records committed, failed sent to DLT)
        acknowledgment.acknowledge();
        log.info("Batch processing complete. Success: {}, Failed: {}", 
            records.size() - failedRecords.size(), failedRecords.size());
    }
}
```

**This approach:**

- Processes all records concurrently
- Tracks failures individually
- Sends failed records to DLT
- Acknowledges the batch once all processing attempts complete

**Trade-off:** More complex but provides better throughput. Failed messages go to DLT instead of reprocessing entire batch.

#### 4.4.4. Retry Strategy for Async Batch Processing

Implement retry logic before sending to DLT to handle transient failures.

**With retry mechanism:**

```java
@Service
public class RetryingAsyncBatchListener {
    
    private static final Logger log = LoggerFactory.getLogger(RetryingAsyncBatchListener.class);
    private static final int MAX_RETRY_ATTEMPTS = 3;
    
    @Autowired
    private ExecutorService kafkaProcessorPool;
    
    @Autowired
    private OrderProcessor orderProcessor;
    
    @Autowired
    private KafkaTemplate<String, Order> kafkaTemplate;
    
    @KafkaListener(
        topics = "orders",
        containerFactory = "batchAsyncFactory"
    )
    public void processBatch(
            List<ConsumerRecord<String, Order>> records,
            Acknowledgment acknowledgment) {
        
        log.info("Processing batch of {} records", records.size());
        
        List<CompletableFuture<RecordResult>> futures = records.stream()
            .map(record -> processWithRetry(record))
            .collect(Collectors.toList());
        
        // Wait for all to complete
        CompletableFuture<Void> allOf = CompletableFuture.allOf(
            futures.toArray(new CompletableFuture[0])
        );
        
        try {
            allOf.get(120, TimeUnit.SECONDS);
        } catch (Exception e) {
            log.error("Batch processing failed", e);
            // Don't acknowledge → batch redelivered
            return;
        }
        
        // Process results
        List<RecordResult> results = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
        
        // Send failed records to DLT after retries exhausted
        results.stream()
            .filter(result -> !result.isSuccess())
            .forEach(result -> sendToDlt(result.getRecord(), result.getException()));
        
        // Acknowledge batch
        acknowledgment.acknowledge();
        
        long successCount = results.stream().filter(RecordResult::isSuccess).count();
        long failureCount = results.size() - successCount;
        log.info("Batch complete. Success: {}, Failed: {}", successCount, failureCount);
    }
    
    private CompletableFuture<RecordResult> processWithRetry(
            ConsumerRecord<String, Order> record) {
        
        return CompletableFuture.supplyAsync(() -> {
            Exception lastException = null;
            
            for (int attempt = 1; attempt <= MAX_RETRY_ATTEMPTS; attempt++) {
                try {
                    orderProcessor.process(record.value());
                    return new RecordResult(record, true, null);
                } catch (Exception e) {
                    lastException = e;
                    log.warn("Attempt {}/{} failed for record offset {}: {}", 
                        attempt, MAX_RETRY_ATTEMPTS, record.offset(), e.getMessage());
                    
                    if (attempt < MAX_RETRY_ATTEMPTS) {
                        // Exponential backoff
                        try {
                            Thread.sleep((long) Math.pow(2, attempt) * 1000);
                        } catch (InterruptedException ie) {
                            Thread.currentThread().interrupt();
                            return new RecordResult(record, false, ie);
                        }
                    }
                }
            }
            
            // All retries exhausted
            return new RecordResult(record, false, lastException);
            
        }, kafkaProcessorPool);
    }
    
    private void sendToDlt(ConsumerRecord<String, Order> record, Exception exception) {
        try {
            ProducerRecord<String, Order> dltRecord = new ProducerRecord<>(
                record.topic() + ".DLT",
                record.partition(),
                record.key(),
                record.value()
            );
            
            // Add error headers
            dltRecord.headers().add("original-topic", record.topic().getBytes());
            dltRecord.headers().add("original-offset", 
                String.valueOf(record.offset()).getBytes());
            dltRecord.headers().add("error-message", 
                exception.getMessage().getBytes());
            dltRecord.headers().add("error-class", 
                exception.getClass().getName().getBytes());
            
            kafkaTemplate.send(dltRecord);
            log.info("Sent to DLT: offset={}", record.offset());
            
        } catch (Exception e) {
            log.error("Failed to send to DLT", e);
        }
    }
    
    @Data
    @AllArgsConstructor
    private static class RecordResult {
        private ConsumerRecord<String, Order> record;
        private boolean success;
        private Exception exception;
    }
}
```

**This implementation:**

- Retries each record up to 3 times with exponential backoff
- Processes all records concurrently in thread pool
- Only sends to DLT after all retries exhausted
- Acknowledges batch after all processing complete

#### 4.4.5. Trade-offs Analysis

**Approach 1: All-or-Nothing (Section 4.4.2)**

✅ **Pros:**
- Simple to implement
- No message loss guaranteed
- Maintains batch ordering

❌ **Cons:**
- One failure reprocesses entire batch
- Lower throughput with frequent failures
- Can cause duplicate processing of successful messages

**Best for:** Low failure rates, strict ordering requirements

---

**Approach 2: Finegrained with DLT (Section 4.4.3)**

✅ **Pros:**
- Higher throughput (no reprocessing successful messages)
- Failed messages isolated to DLT
- Better resource utilization

❌ **Cons:**
- More complex code
- Potential ordering issues if replaying from DLT
- Need to monitor and replay DLT messages

**Best for:** High throughput requirements, acceptable to lose ordering for failed messages

---

**Approach 3: Retry with DLT (Section 4.4.4)**

✅ **Pros:**
- Handles transient failures automatically
- Best throughput/reliability balance
- Reduces DLT volume

❌ **Cons:**
- Most complex implementation
- Increased processing latency during retries
- Thread pool can get saturated during failures

**Best for:** Production systems with transient failures (database timeouts, rate limits)

---

**Throughput Comparison:**

| Approach | Throughput | Latency | Complexity | Data Loss Risk |
|----------|-----------|---------|------------|----------------|
| Sync Batch (no thread pool) | Low | Low | Low | None |
| All-or-Nothing Async | Medium-High | Medium | Medium | None |
| Finegrained Async | High | Low | High | None (with proper DLT) |
| Retry + Async | High | Medium | High | None |

**Configuration recommendations:**

```yaml
# For high throughput with async processing
spring:
  kafka:
    consumer:
      max-poll-records: 500  # Larger batches
      fetch-min-size: 1024   # Wait for more data
      fetch-max-wait: 500    # But don't wait too long
    listener:
      ack-mode: manual_immediate
      concurrency: 3  # Multiple container threads

# Thread pool sizing
kafka:
  processor:
    core-threads: 20
    max-threads: 100
    queue-capacity: 2000
```

**Thread pool sizing guidelines:**

- **Core threads:** 2-3x number of Kafka consumer threads
- **Max threads:** Based on downstream service capacity
- **Queue capacity:** `max-poll-records` × `concurrency` × 2

Example: If `max-poll-records=500`, `concurrency=3`, queue should be ~3000.

#### 4.4.6. Monitoring and Observability

When using async batch processing, add metrics to track:

```java
@Service
public class MonitoredAsyncBatchListener {
    
    private final MeterRegistry meterRegistry;
    private final Counter processedCounter;
    private final Counter failedCounter;
    private final Timer processingTimer;
    
    public MonitoredAsyncBatchListener(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.processedCounter = Counter.builder("kafka.messages.processed")
            .tag("type", "batch-async")
            .register(meterRegistry);
        this.failedCounter = Counter.builder("kafka.messages.failed")
            .tag("type", "batch-async")
            .register(meterRegistry);
        this.processingTimer = Timer.builder("kafka.batch.processing.time")
            .register(meterRegistry);
    }
    
    @KafkaListener(topics = "orders", containerFactory = "batchAsyncFactory")
    public void processBatch(
            List<ConsumerRecord<String, Order>> records,
            Acknowledgment acknowledgment) {
        
        processingTimer.record(() -> {
            // Your processing logic here
            processBatchInternal(records, acknowledgment);
        });
    }
    
    private void processBatchInternal(
            List<ConsumerRecord<String, Order>> records,
            Acknowledgment acknowledgment) {
        // Track batch size
        meterRegistry.gauge("kafka.batch.size", records.size());
        
        // Process batch...
        // Update counters based on results
        processedCounter.increment(successCount);
        failedCounter.increment(failureCount);
    }
}
```

**Key metrics to monitor:**

- Batch processing time
- Success/failure rates
- Thread pool utilization
- DLT message count
- Consumer lag

## 5. Deserialization Error Handling

### 5.1. Using ErrorHandlingDeserializer

Deserialization errors are tricky because they happen before your `@KafkaListener` method is invoked. Without special handling, a single bad message stops the entire consumer.

**Solution:** Wrap your deserializer with `ErrorHandlingDeserializer`.

**Configuration:**

```java
@Bean
public ConsumerFactory<String, Order> consumerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    
    // Wrap the actual deserializer
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
    props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);
    props.put(JsonDeserializer.TRUSTED_PACKAGES, "*");
    
    return new DefaultKafkaConsumerFactory<>(props);
}
```

**Or with properties:**

```yaml
spring:
  kafka:
    consumer:
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        spring.json.trusted.packages: "*"
```

### 5.2. Handling Deserialization Failures

With `ErrorHandlingDeserializer`, failed records arrive with `null` value and exception details in headers.

**Record listener:**

```java
@KafkaListener(topics = "orders")
public void listen(ConsumerRecord<String, Order> record) {
    if (record.value() == null) {
        // Deserialization failed
        throw new DeserializationException("Cannot deserialize", null, null);
    }
    
    processOrder(record.value());
}
```

**Batch listener:**

```java
@KafkaListener(topics = "orders", containerFactory = "batchFactory")
public void listenBatch(List<ConsumerRecord<String, Order>> records) {
    for (ConsumerRecord<String, Order> record : records) {
        if (record.value() == null) {
            // Deserialization failed → send to DLT
            throw new BatchListenerFailedException(
                "Deserialization failed", 
                record
            );
        }
        processOrder(record.value());
    }
}
```

**Important:** For batch listeners, you must manually check for `null` values. There's no automatic detection like with record listeners.

### 5.3. Conversion Errors

Starting with version 2.8, batch listeners can handle conversion errors when using a `MessageConverter`.

**Example:**

```java
@KafkaListener(topics = "orders")
public void listen(
        List<Order> orders,
        @Header(KafkaHeaders.CONVERSION_FAILURES) List<ConversionException> exceptions) {
    
    for (int i = 0; i < orders.size(); i++) {
        if (orders.get(i) == null && exceptions.get(i) != null) {
            throw new BatchListenerFailedException(
                "Conversion error", 
                exceptions.get(i), 
                i
            );
        }
        processOrder(orders.get(i));
    }
}
```

## 6. Backoff Strategies

### 6.1. Fixed Backoff

Retries at fixed intervals.

```java
// 3 retries with 1 second between each
new FixedBackOff(1000L, 2L)
```

### 6.2. Exponential Backoff

Retries with exponentially increasing delays.

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, Order> template) {
    ExponentialBackOffWithMaxRetries backOff = new ExponentialBackOffWithMaxRetries(6);
    backOff.setInitialInterval(1000L);    // Start with 1 second
    backOff.setMultiplier(2.0);           // Double each time
    backOff.setMaxInterval(10000L);       // Cap at 10 seconds
    
    // Retry after: 1s, 2s, 4s, 8s, 10s, 10s
    
    DeadLetterPublishingRecoverer recoverer = 
        new DeadLetterPublishingRecoverer(template);
    
    return new DefaultErrorHandler(recoverer, backOff);
}
```

### 6.3. Custom Backoff Based on Exception

Different backoff strategies for different errors:

```java
DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, defaultBackOff);

handler.setBackOffFunction((record, exception) -> {
    if (exception.getCause() instanceof TransientException) {
        // Retry quickly for transient errors
        return new FixedBackOff(500L, 5L);
    } else if (exception.getCause() instanceof RateLimitException) {
        // Wait longer for rate limit errors
        return new FixedBackOff(5000L, 3L);
    }
    // Use default backoff for other cases
    return null;
});
```

## 7. Common Pitfalls and Best Practices

### 7.1. Losing Messages with Async Processing

**Problem:** When processing messages asynchronously with a custom thread pool, the `@KafkaListener` method returns immediately. If using `BATCH` or `RECORD` ack mode, the offset is committed before async processing completes.

```java
// DON'T DO THIS
@KafkaListener(topics = "orders")
public void listen(Order order) {
    executorService.submit(() -> {
        // Process asynchronously
        processOrder(order);  // If this fails, message is lost
    });
    // Method returns → offset committed before processing!
}
```

**Solution:** Use `MANUAL_IMMEDIATE` ack mode:

```java
@KafkaListener(
    topics = "orders",
    containerFactory = "manualFactory"
)
public void listen(Order order, Acknowledgment ack) {
    executorService.submit(() -> {
        try {
            processOrder(order);
            ack.acknowledge();  // Commit only after success
        } catch (Exception e) {
            log.error("Failed", e);
            // Don't acknowledge → message will be redelivered
        }
    });
}
```

### 7.2. Message Duplicates After Restart

**Problem:** Consumer crashes after processing but before committing offset → message gets reprocessed.

This is normal with at-least-once delivery. Your processing logic should be **idempotent**.

**Solution — Idempotent processing:**

```java
@KafkaListener(topics = "orders")
public void listen(Order order) {
    // Use database constraints or unique checks
    try {
        orderRepository.insertIfNotExists(order);
    } catch (DuplicateKeyException e) {
        log.info("Order {} already processed, skipping", order.getId());
        // Don't throw exception → acknowledge and move on
    }
}
```

Or track processed message IDs in a separate table:

```java
@Transactional
public void processOrder(Order order) {
    if (processedMessageRepository.exists(order.getMessageId())) {
        log.info("Already processed message {}", order.getMessageId());
        return;
    }
    
    // Process the order
    orderService.createOrder(order);
    
    // Mark as processed
    processedMessageRepository.save(new ProcessedMessage(order.getMessageId()));
}
```

### 7.3. Forgetting to Configure DLT Topics

**Problem:** DefaultErrorHandler tries to send to DLT, but topic doesn't exist → error on error.

**Solution:** Create DLT topics ahead of time with appropriate partition count:

```bash
# Original topic has 10 partitions
kafka-topics.sh --create \
  --topic orders.DLT \
  --partitions 10 \
  --replication-factor 3
```

Or configure to create dynamically:

```java
@Bean
public NewTopic ordersDlt() {
    return TopicBuilder
        .name("orders.DLT")
        .partitions(10)
        .replicas(3)
        .build();
}
```

### 7.4. Not Setting Immediate Stop for Long Processing

**Problem:** Graceful shutdown waits for entire batch to complete. If each message takes 30 seconds, shutdown can take minutes.

**Solution:**

```yaml
spring:
  kafka:
    listener:
      ack-mode: RECORD
      immediate-stop: true  # Stop after current record
```

### 7.5. Batch Listener without BatchListenerFailedException

**Problem:** Throwing generic exception from batch listener retries the entire batch, can't identify which record failed.

**Bad:**

```java
@KafkaListener(topics = "orders", containerFactory = "batchFactory")
public void processBatch(List<Order> orders) {
    for (Order order : orders) {
        processOrder(order);  // Exception loses context
    }
}
```

**Good:**

```java
@KafkaListener(topics = "orders", containerFactory = "batchFactory")
public void processBatch(List<Order> orders) {
    for (int i = 0; i < orders.size(); i++) {
        try {
            processOrder(orders.get(i));
        } catch (Exception e) {
            throw new BatchListenerFailedException("Failed", e, i);
        }
    }
}
```

## 8. Production Configuration Example

Here's a solid production-ready configuration combining everything:

```java
@Configuration
public class KafkaErrorHandlingConfig {
    
    @Bean
    public ConsumerFactory<String, Order> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        
        // Wrap deserializer for error handling
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);
        props.put(JsonDeserializer.TRUSTED_PACKAGES, "com.example.model");
        
        // Disable auto-commit (Spring Kafka manages it)
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        
        return new DefaultKafkaConsumerFactory<>(props);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Order> kafkaListenerContainerFactory(
            ConsumerFactory<String, Order> consumerFactory,
            KafkaTemplate<String, Order> kafkaTemplate) {
        
        ConcurrentKafkaListenerContainerFactory<String, Order> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        
        // Commit after each record for minimal duplicates
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.RECORD);
        
        // Stop immediately on shutdown
        factory.getContainerProperties().setStopImmediate(true);
        
        // Configure DLT error handler
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            kafkaTemplate,
            (record, ex) -> {
                // Custom DLT routing based on exception
                if (ex.getCause() instanceof ValidationException) {
                    return new TopicPartition(record.topic() + ".validation.DLT", -1);
                }
                return new TopicPartition(record.topic() + ".DLT", record.partition());
            }
        );
        
        // Exponential backoff with max retries
        ExponentialBackOffWithMaxRetries backOff = new ExponentialBackOffWithMaxRetries(5);
        backOff.setInitialInterval(1000L);
        backOff.setMultiplier(2.0);
        backOff.setMaxInterval(30000L);
        
        DefaultErrorHandler errorHandler = new DefaultErrorHandler(recoverer, backOff);
        
        // Don't retry validation errors
        errorHandler.addNotRetryableExceptions(ValidationException.class);
        
        // Reset retry state if exception type changes
        errorHandler.setResetStateOnExceptionChange(true);
        
        factory.setCommonErrorHandler(errorHandler);
        
        return factory;
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Order> batchFactory(
            ConsumerFactory<String, Order> consumerFactory,
            KafkaTemplate<String, Order> kafkaTemplate) {
        
        ConcurrentKafkaListenerContainerFactory<String, Order> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setBatchListener(true);
        
        // Similar error handler for batch processing
        DeadLetterPublishingRecoverer recoverer = 
            new DeadLetterPublishingRecoverer(kafkaTemplate);
        
        DefaultErrorHandler errorHandler = new DefaultErrorHandler(
            recoverer,
            new FixedBackOff(1000L, 3L)
        );
        
        factory.setCommonErrorHandler(errorHandler);
        
        return factory;
    }
    
    // Create DLT topics automatically
    @Bean
    public NewTopic ordersDlt() {
        return TopicBuilder
            .name("orders.DLT")
            .partitions(10)
            .replicas(3)
            .build();
    }
    
    @Bean
    public NewTopic ordersValidationDlt() {
        return TopicBuilder
            .name("orders.validation.DLT")
            .partitions(10)
            .replicas(3)
            .build();
    }
}
```

**application.yml:**

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: order-service
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        spring.json.trusted.packages: com.example.model
    listener:
      ack-mode: record
      immediate-stop: true

logging:
  level:
    org.springframework.kafka: DEBUG
```

## 9. Final Thoughts

Error handling in Spring Kafka requires understanding how offsets, acknowledgments, retries, and dead letter topics work together.

**Key takeaways:**

- Use `RECORD` ack mode for minimal duplicates, `BATCH` for higher throughput
- Always configure `DeadLetterPublishingRecoverer` for production
- Make your processing logic idempotent to handle duplicates
- Use `ErrorHandlingDeserializer` to prevent deserialization errors from blocking consumers
- For batch listeners, always throw `BatchListenerFailedException` with the failed record
- Use `MANUAL_IMMEDIATE` ack mode for async processing with custom thread pools
- For maximum throughput, process batches asynchronously with `CompletableFuture` — track all tasks before acknowledging
- Choose async batch processing strategy based on requirements: all-or-nothing (simple), finegrained (high throughput), or retry-based (best balance)
- Configure appropriate backoff strategies based on your error types
- Create DLT topics ahead of time with correct partition counts
- Size thread pools based on consumer concurrency: core threads = 2-3x consumer threads, queue = max-poll-records × concurrency × 2
- Monitor async processing metrics: processing time, success/failure rates, thread pool utilization, consumer lag

Spring Kafka's error handling is flexible enough to handle most real-world scenarios. The key is understanding the trade-offs between throughput, latency, and message delivery guarantees. When pushing for maximum throughput with async batch processing, carefully balance implementation complexity against reliability requirements.

## 10. References

- <https://docs.spring.io/spring-kafka/reference/kafka/annotation-error-handling.html>
- <https://piotrminkowski.com/2024/03/11/kafka-offset-with-spring-boot/>
- <https://runebook.dev/en/docs/spring_boot/application-properties/application-properties.integration.spring.kafka.listener.async-acks>
