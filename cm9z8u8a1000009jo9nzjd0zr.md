---
title: "Can I send 1million logs per second to kafka locally"
datePublished: Sun Apr 27 2025 06:03:34 GMT+0000 (Coordinated Universal Time)
cuid: cm9z8u8a1000009jo9nzjd0zr
slug: can-i-send-1million-logs-per-second-to-kafka-locally
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1745733700013/eda4378c-c9a6-492c-bf7d-030d0cef1c48.png
tags: optimization, python, golang, logging, kafka

---

%[https://github.com/Mukund-Tandon/Logger/tree/main/logingressor/testscript/kafka-test-scripts] 

## Test Setup

To properly evaluate our log ingestor's performance, I've created a test environment that simulates high-volume log production. This setup allows us to measure throughput, latency, and resource utilization under various loads.

### Kafka Infrastructure

My test environment uses a simple Kafka cluster with two brokers, managed through Docker Compose:

```yaml
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.4
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - 2181:2181

  kafka-1:
    image: confluentinc/cp-kafka:7.4.4
    depends_on:
      - zookeeper
    ports:
      - 29092:29092
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,PLAINTEXT_HOST://0.0.0.0:29092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-1:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  kafka-2:
    image: confluentinc/cp-kafka:7.4.4
    depends_on:
      - zookeeper
    ports:
      - 39092:39092
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,PLAINTEXT_HOST://0.0.0.0:39092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-2:9092,PLAINTEXT_HOST://localhost:39092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      - kafka-1
      - kafka-2
    ports:
      - 8081:8080
    environment:
      KAFKA_CLUSTERS_0_NAME: local-kafka
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka-1:9092,kafka-2:9092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181
      DYNAMIC_CONFIG_ENABLED: 'true'
```

This configuration provides:

* A single ZooKeeper instance for cluster coordination
    
* Two Kafka brokers (kafka-1 and kafka-2) with externally accessible ports
    
* Kafka UI for visual monitoring of topics and consumer groups
    

The Kafka brokers are configured with a replication factor of 1, which is sufficient for testing purposes but would be increased in a production environment.

### Python Test Script

To generate load, I've created a multi-threaded Python script that can produce logs at various rates. This script is an enhanced version of my original producer:

```python

from kafka import KafkaProducer
from kafka.errors import KafkaError
import json
import time
import datetime
import random
import threading

class KafkaMessageProducer:
    def __init__(self, bootstrap_servers, topic_name):
        self.topic_name = topic_name
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None,
            acks=1,
            retries=3,
            max_in_flight_requests_per_connection=20,
            linger_ms=50, 
            batch_size=128 * 1024, 
        )
        # Metrics
        self.metrics = {
            "sent_count": 0,
            "error_count": 0,
            "start_time": None,
            "end_time": None
        }
        self.metrics_lock = threading.Lock()

    def send_message(self, message):
        self.producer.send(self.topic_name, value=message)
        try:
            with self.metrics_lock:
                self.metrics["sent_count"] += 1
            return True
        except KafkaError as e:
            with self.metrics_lock:
                self.metrics["error_count"] += 1
            return False

    def send_messages_in_thread(self, messages, thread_id):
        for i, msg in enumerate(messages):
            success = self.send_message(msg)

    def send_messages_threaded(self, messages, num_threads=10):
        self.metrics["start_time"] = time.time()
        
        msgs_per_thread = len(messages) // num_threads
        if msgs_per_thread == 0:
            msgs_per_thread = 1
            num_threads = min(num_threads, len(messages))
        
        threads = []
        for i in range(num_threads):
            start_idx = i * msgs_per_thread
            end_idx = start_idx + msgs_per_thread if i < num_threads - 1 else len(messages)
            thread_messages = messages[start_idx:end_idx]
            
            thread = threading.Thread(
                target=self.send_messages_in_thread,
                args=(thread_messages, i)
            )
            threads.append(thread)
            thread.start()
        
        for thread in threads:
            thread.join()
        
        self.metrics["end_time"] = time.time()
        duration = self.metrics["end_time"] - self.metrics["start_time"]
        rate = self.metrics["sent_count"] / duration if duration > 0 else 0
        
        print(f"Completed sending {self.metrics['sent_count']} messages in {duration:.2f} seconds")
        print(f"Throughput: {rate:.2f} messages/second")
        print(f"Errors: {self.metrics['error_count']}")
        
        return {
            "messages_sent": self.metrics["sent_count"],
            "errors": self.metrics["error_count"],
            "duration_seconds": duration,
            "throughput": rate
        }

    def close(self):
        self.producer.close()

def generate_log_message(i):
    log_levels = ["INFO", "WARN", "ERROR", "DEBUG"]
    return {
        'Timestamp': datetime.datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%S.%fZ"),
        'Level': random.choice(log_levels),
        'Message': f'Test log message {i}',
        'ResourceID': f'resource-{i % 10}'  # Using modulo to repeat some resource IDs
    }

def generate_messages(count):
    return [generate_log_message(i) for i in range(count)]

if __name__ == "__main__":
    import argparse
    
    parser = argparse.ArgumentParser(description='Kafka log producer')
    parser.add_argument('--brokers', default='localhost:29092,localhost:39092', help='Kafka brokers')
    parser.add_argument('--topic', default='logs', help='Kafka topic name')
    parser.add_argument('--messages', type=int, default=2000000, help='Number of messages to send')
    parser.add_argument('--threads', type=int, default=20, help='Number of threads')
    
    args = parser.parse_args()
    
    bootstrap_servers = args.brokers.split(',')
    topic_name = args.topic
    num_messages = args.messages
    num_threads = args.threads
    
    print(f"Starting Kafka producer with {num_threads} threads to send {num_messages} messages")
    print(f"Connecting to Kafka at {bootstrap_servers}, topic: {topic_name}")
    
    producer = KafkaMessageProducer(bootstrap_servers, topic_name)
    
    try:
        messages = generate_messages(num_messages)
        producer.send_messages_threaded(messages, num_threads=num_threads)
        
    except KeyboardInterrupt:
        print("Interrupted by user, shutting down...")
    finally:
        producer.close()
```

The script includes several key features:

* Multi-threaded message production for high throughput
    
* Metrics collection (sent count, error count, throughput)
    
* Configurable message count and thread count
    
* Consistent log format matching our ingestor's expectations
    

## Test results after running this

After implementing our initial Python script, I ran several tests to measure its performance. The results were underwhelming, showing we needed significant optimizations to approach our target throughput.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745064881179/e8704375-c925-46a3-8732-e05ac5430d6b.png align="center")

### Adding Compression

To improve throughput, I first added data compression to the Kafka producer configuration. Compression reduces the size of messages sent over the network, potentially allowing more messages to be transmitted in the same time period:

```python
self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None,
            acks=1,
            retries=3,
            max_in_flight_requests_per_connection=20,
            linger_ms=50, 
            batch_size=128 * 1024, 
            compression_type='snappy' 
       )
```

The `compression_type='snappy'` parameter enabled Snappy compression, which offers a good balance between compression ratio and CPU usage. After this change, the number of messages per second increased slightly, but we were still far from our target of 1 million messages per second.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745065010145/2b6f3d60-0377-406a-bbf0-8e56769fae28.png align="center")

### Increasing Batch Size and Adjusting Timing

For my next optimization attempt, I decided to increase the batch size to 256KB and adjust several other parameters:

```python
self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None,
            acks=1,
            retries=3,
            max_in_flight_requests_per_connection=40,
            linger_ms=100, 
            batch_size=256 * 1024, 
            compression_type='snappy'
        )
```

The key changes were:

* Doubling `max_in_flight_requests_per_connection` from 20 to 40, allowing more concurrent requests
    
* Increasing `linger_ms` from 50ms to 100ms, giving the producer more time to batch messages
    
* Doubling the `batch_size` from 128KB to 256KB, allowing larger message batches
    

Unfortunately, these changes didn't result in a significant performance improvement. The throughput remained roughly the same, indicating that these parameters weren't the main bottleneck.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745065262050/1242ddad-0420-43a3-9eeb-2c9371f9a8af.png align="center")

### Trying Alternative Compression Algorithm

For my third attempt, I switched to a different compression algorithm (LZ4), further increased the batch size, and adjusted other parameters:

```python
self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None,
            acks=1,
            retries=3,
            max_in_flight_requests_per_connection=60,
            linger_ms=50, 
            batch_size=512 * 1024, 
            compression_type='lz4'
        )
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745065632049/d6ed1fe6-8aa3-4813-a36e-357c36e3d004.png align="center")

This configuration:

* Changed the compression algorithm to LZ4, which can be faster than Snappy
    
* Further increased `max_in_flight_requests_per_connection` to 60
    
* Returned `linger_ms` to 50ms to reduce waiting time
    
* Doubled the `batch_size` again to 512KB
    

This configuration still didn't yield the dramatic improvement I was looking for, suggesting we had reached the performance ceiling for the Python Kafka producer.

### Experimenting with Thread Count

Since modifying the Kafka producer configuration parameters wasn't providing significant gains, I decided to experiment with the number of threads. I tried various thread c  
Increasing from 20 to 40 threads

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745065780674/ba0e4482-1dbe-4487-9eee-330fe0cd6834.png align="center")

Reducing from 40 to 30 threads

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745065985884/f84e358c-df8a-4fa3-b4dc-73898c94366d.png align="center")

Further reducing from 30 to 25 threads

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745066085363/e2cf3d8f-ac48-4441-97e7-eddea2d5f00f.png align="center")

These experiments yielded only marginal improvements. I concluded that we had likely reached the performance limit of what was possible with the Python implementation.

### Moving to Go for Higher Performance

At this point, I decided that a more fundamental change was needed. Rather than continue tweaking the Python implementation, I rewrote the entire script in Go, a language known for its excellent concurrency model and performance characteristics. This decision would prove to be the breakthrough we needed to achieve our performance goals.

```go
package main

import (
	"context"
	"encoding/json"
	"flag"
	"fmt"
	"math/rand"
	"strings"
	"sync"
	"time"

	"github.com/segmentio/kafka-go"
)

type LogMessage struct {
	Timestamp  string `json:"Timestamp"`
	Level      string `json:"Level"`
	Message    string `json:"Message"`
	ResourceID string `json:"ResourceID"`
}

type KafkaMessageProducer struct {
	writer     *kafka.Writer
	metrics    struct {
		sentCount  int
		errorCount int
		startTime  time.Time
		endTime    time.Time
	}
	metricsLock sync.Mutex
}

func NewKafkaMessageProducer(brokers []string, topic string) *KafkaMessageProducer {
	producer := &KafkaMessageProducer{}

	producer.writer = &kafka.Writer{
		Addr:         kafka.TCP(brokers...),
		Topic:        topic,
		BatchSize:    256 * 1024, // 256KB batch size
		BatchTimeout: 50 * time.Millisecond,
		Async:        true,
		RequiredAcks: kafka.RequireOne,
		Compression:  kafka.Snappy,
		MaxAttempts:  3,
	}

	return producer
}

func (p *KafkaMessageProducer) SendMessage(ctx context.Context, message interface{}) bool {
	
	msgBytes, err := json.Marshal(message)
	if err != nil {
		p.metricsLock.Lock()
		p.metrics.errorCount++
		p.metricsLock.Unlock()
		return false
	}

	err = p.writer.WriteMessages(ctx, kafka.Message{
		Value: msgBytes,
	})

	if err != nil {
		p.metricsLock.Lock()
		p.metrics.errorCount++
		p.metricsLock.Unlock()
		return false
	}

	p.metricsLock.Lock()
	p.metrics.sentCount++
	p.metricsLock.Unlock()
	return true
}

func (p *KafkaMessageProducer) SendMessagesInGoroutine(ctx context.Context, messages []LogMessage, goroutineID int) {
	for _, msg := range messages {
		p.SendMessage(ctx, msg)
	}
}

func (p *KafkaMessageProducer) SendMessagesParallel(ctx context.Context, messages []LogMessage, numGoroutines int) map[string]interface{} {
	p.metrics.startTime = time.Now()

	msgsPerGoroutine := len(messages) / numGoroutines
	if msgsPerGoroutine == 0 {
		msgsPerGoroutine = 1
		numGoroutines = min(numGoroutines, len(messages))
	}

	var wg sync.WaitGroup
	for i := 0; i < numGoroutines; i++ {
		wg.Add(1)

		startIdx := i * msgsPerGoroutine
		endIdx := startIdx + msgsPerGoroutine
		if i == numGoroutines-1 {
			endIdx = len(messages)
		}

		goroutineMessages := messages[startIdx:endIdx]

		go func(id int, msgs []LogMessage) {
			defer wg.Done()
			p.SendMessagesInGoroutine(ctx, msgs, id)
		}(i, goroutineMessages)
	}

	wg.Wait()

	p.metrics.endTime = time.Now()
	duration := p.metrics.endTime.Sub(p.metrics.startTime).Seconds()
	rate := float64(p.metrics.sentCount) / duration

	fmt.Printf("Completed sending %d messages in %.2f seconds\n", p.metrics.sentCount, duration)
	fmt.Printf("Throughput: %.2f messages/second\n", rate)
	fmt.Printf("Errors: %d\n", p.metrics.errorCount)

	return map[string]interface{}{
		"messages_sent":     p.metrics.sentCount,
		"errors":            p.metrics.errorCount,
		"duration_seconds":  duration,
		"throughput":        rate,
	}
}

func (p *KafkaMessageProducer) Close() {
	p.writer.Close()
}

func GenerateLogMessage(i int) LogMessage {
	logLevels := []string{"INFO", "WARN", "ERROR", "DEBUG"}
	return LogMessage{
		Timestamp:  time.Now().UTC().Format(time.RFC3339Nano),
		Level:      logLevels[rand.Intn(len(logLevels))],
		Message:    fmt.Sprintf("Test log message %d", i),
		ResourceID: fmt.Sprintf("resource-%d", i%10),
	}
}

func GenerateMessages(count int) []LogMessage {
	messages := make([]LogMessage, count)
	for i := 0; i < count; i++ {
		messages[i] = GenerateLogMessage(i)
	}
	return messages
}

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}

func main() {
	brokers := flag.String("brokers", "localhost:29092,localhost:39092", "Kafka brokers")
	topic := flag.String("topic", "logs", "Kafka topic name")
	numMessages := flag.Int("messages", 2000000, "Number of messages to send")
	numGoroutines := flag.Int("goroutines", 25, "Number of goroutines")
	flag.Parse()

	brokersList := strings.Split(*brokers, ",")

	fmt.Printf("Starting Kafka producer with %d goroutines to send %d messages\n", *numGoroutines, *numMessages)
	fmt.Printf("Connecting to Kafka at %v, topic: %s\n", brokersList, *topic)

	producer := NewKafkaMessageProducer(brokersList, *topic)
	ctx := context.Background()

	messages := GenerateMessages(*numMessages)

	defer producer.Close()
	producer.SendMessagesParallel(ctx, messages, *numGoroutines)
}
```

Switching to Go immediately delivered a dramatic 20x performance improvement over Python. But to reach our ambitious goal of 1 million messages per second, I needed to push the implementation even further through careful analysis and targeted optimizations.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745104885562/c17d22be-0f42-4072-84a2-d8667d8ecd94.png align="center")

### Optimizing the Kafka Writer Configuration

The heart of my Go implementation was the `kafka.Writer` configuration. After multiple iterations, I settled on this highly optimized setup:

```go
goproducer.writer = &kafka.Writer{
    Addr:            kafka.TCP(brokers...),
    Topic:           topic,
    BatchSize:       4000,
    BatchBytes:      128 * 1024,
    BatchTimeout:    50 * time.Millisecond,
    Async:           true,
    RequiredAcks:    kafka.RequireOne,
    Compression:     kafka.Snappy,
    MaxAttempts:     3,
    WriteBackoffMin: 10 * time.Millisecond,
    WriteBackoffMax: 500 * time.Millisecond,
}
```

Let's break down the key parameters that made this configuration so effective:

* **BatchSize: 4000** - This setting dramatically increased the number of messages grouped together in a single batch from the default of 100. Since each log message was approximately 124 bytes, this allowed much more efficient network utilization without exceeding memory constraints.
    
* **BatchBytes: 128KB** - This parameter sets the maximum size of a batch in bytes. While 4000 messages would theoretically require about 496KB (4000 × 124 bytes), I found that 128KB worked better in practice due to compression and the actual distribution of message sizes.
    
* **BatchTimeout: 50ms** - This setting created a balance between latency and throughput. The producer would wait up to 50ms to accumulate a full batch, but would send whatever it had collected after this timeout expired.
    
* **Async: true** - This crucial setting allowed the writer to send messages asynchronously, preventing the producer from blocking while waiting for acknowledgments from Kafka.
    
* **Compression: Snappy** - After testing multiple compression algorithms, Snappy provided the best balance between compression ratio and CPU overhead for our specific message format.
    
* **WriteBackoffMin/Max: 10ms/500ms** - These parameters control how long the writer waits before retrying after a failed write. The relatively low minimum backoff of 10ms allowed faster recovery from transient issues.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745106850819/f143bf40-07e0-4021-bfbf-13d81b979468.png align="center")

### Eliminating Lock Contention

One of the most significant optimizations was replacing mutex locks with atomic operations. In high-throughput applications, lock contention becomes a major bottleneck as thousands of operations per second compete for the same locks.

Instead of using mutex-protected counters:

```go

p.metricsLock.Lock()
p.metrics.sentCount++
p.metricsLock.Unlock()
```

I switched to atomic operations:

```go

atomic.AddInt64(&p.metrics.sentCount, 1)
```

This eliminated wait times for lock acquisition, allowing goroutines to increment counters without blocking each other.

### Pre-serializing Messages

Another crucial optimization was pre-serializing all log messages during the preparation phase rather than during sending:

```go
go// Generate all messages with pre-serialization
messages := make([]kafka.Message, numMessages)
for i := 0; i < numMessages; i++ {
    logMsg := generateLogMessage(i)
    jsonData, _ := json.Marshal(logMsg)
    messages[i] = kafka.Message{Value: jsonData}
}

// Later, during send, the messages are already serialized
p.writer.WriteMessages(ctx, messages...)
```

This moved the JSON serialization overhead out of the critical path, allowing the send operations to run at maximum efficiency.

Pre sterializing provided the maximum jump-

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745121401399/2dbe35a7-448c-4bd2-b34e-902744fdfb4f.png align="center")

### Aligning with Hardware Capabilities

The final breakthrough came when I aligned the application with my hardware capabilities. Through experimentation, I discovered that the optimal number of goroutines matched exactly with my CPU core count. This minimized context switching overhead and made the most efficient use of available processing power.

To maximize available resources, I closed nearly all other applications running on my laptop, leaving only my code editor and Docker Desktop active. This reduced competition for CPU, memory, and I/O resources.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745121561784/40ee53a8-4291-4d62-be4e-715decfff104.png align="center")

With these carefully tuned optimizations working in concert, I finally watched the throughput counter climb past the 1 million messages per second mark—a genuine achievement that demonstrated the power of Go's concurrency model when properly optimized.

This experience reinforced an important lesson in performance engineering: while high-level changes (like switching programming languages) can provide order-of-magnitude improvements, reaching extreme performance targets often requires understanding and optimizing at multiple levels simultaneously—from hardware utilization to memory management to eliminating even the smallest inefficiencies in the critical path.

## Conclusion

This journey from 25,000 to 1 million messages per second demonstrates how strategic language selection, targeted optimizations, and hardware alignment can collectively overcome seemingly impossible performance barriers. The 40x throughput improvement wasn't achieved through any single change, but through methodical analysis and improvements across multiple dimensions.

The key lessons from this experience extend beyond Kafka producers. When building high-performance systems, consider that:

1. Language selection matters tremendously for performance-critical applications
    
2. Understanding your message characteristics enables precise batching optimizations
    
3. Lock contention often becomes the bottleneck in concurrent high-throughput systems
    
4. Aligning with hardware capabilities (particularly CPU core count) can unlock maximum performance
    
5. Moving work out of critical paths (like pre-serializing messages) pays significant dividends
    

These principles apply broadly to distributed systems engineering and can help you achieve breakthrough performance in your own applications. Sometimes the path to 10x improvement isn't a single clever optimization but a systematic approach to eliminating every source of inefficiency.