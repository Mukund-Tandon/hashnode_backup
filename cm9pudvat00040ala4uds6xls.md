---
title: "blog 2 dsdssd - uu"
datePublished: Sun Apr 20 2025 16:09:01 GMT+0000 (Coordinated Universal Time)
cuid: cm9pudvat00040ala4uds6xls
slug: blog-2-dsdssd-uu

---

## est Setup

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

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745064881179/e8704375-c925-46a3-8732-e05ac5430d6b.png align="center")

to get more throught we can try icreasing the I added one more config to kafka producer , I compressed the data before sending-

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

After this the number of messages persecond di increased a little but we were still far away fomr our goal-

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745065010145/2b6f3d60-0377-406a-bbf0-8e56769fae28.png align="center")

We can try to increase batch size 20 256kb and linger ms to 100ms and max inflight connect to 40

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

but it dididint did much iprovemenet

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745065262050/1242ddad-0420-43a3-9eeb-2c9371f9a8af.png align="center")

trying a new compressin algo and increase nflight connection and decreases linger\_ms-

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

I think we have reached the cap kafka procuder configs so what if I increase number of threads that should work-  
Onincreasig the thread fro 20 to 40

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745065780674/ba0e4482-1dbe-4487-9eee-330fe0cd6834.png align="center")

40 - 30

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745065985884/f84e358c-df8a-4fa3-b4dc-73898c94366d.png align="center")

30 -25

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745066085363/e2cf3d8f-ac48-4441-97e7-eddea2d5f00f.png align="center")

Rewriting the script in go

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

and the speed just increased my 20X-

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745104885562/c17d22be-0f42-4072-84a2-d8667d8ecd94.png align="center")

Now tryng some more optimizations-  
**Timestamp**: Using RFC3339Nano format produces a string like "2025-04-20T15:23:45.123456789Z"

* Approximately 30 bytes
    

1. **Level**: One of "INFO", "WARN", "ERROR", "DEBUG"
    
    * Between 4-5 bytes
        
2. **Message**: "Test log message X" where X is the message number
    
    * "Test log message " is 16 bytes
        
    * The number could be anywhere from 1 digit to many digits, but let's assume an average of 2-3 digits
        
    * About 18-19 bytes total
        
3. **ResourceID**: "resource-X" where X is between 0-9
    
    * "resource-" is 9 bytes
        
    * The number is 1 digit (0-9)
        
    * About 10 bytes total
        
4. **JSON structure overhead**:
    
    * Field names with quotes and colons: ~55 bytes
        
    * Braces, commas: ~5 bytes
        
    * Total overhead: ~60 bytes
        

Adding these up: 30 + 5 + 19 + 10 + 60 = approximately 124 bytes per message

default batch size is 100 mesaegs so in one batch 12400bytes data is send

I can increase my batch size to 4000 and my bmax batch size buffer to 1mb

I alos decreased WriteBackoffMin which is smallest amount of time the writer waits before it attempts to write a batch of messages

We can see more improvement-

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745106850819/f143bf40-07e0-4021-bfbf-13d81b979468.png align="center")

After few more optimization like using atomicvariabel instead of aquiring mutex locks and pre sterializing my logs,this did a significant inprovement helped reaching before only so that I dont strialize them later while sending -

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745121401399/2dbe35a7-448c-4bd2-b34e-902744fdfb4f.png align="center")

after further tring to tne the number of goroutines and batch size , I realized keeping number of gourutine s equal to my actual number of cores hepled the lot and gave more optimal results- also cloesed almost all other running application on my lapto just my code editor and coker desktop was runnig-

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745121561784/40ee53a8-4291-4d62-be4e-715decfff104.png align="center")

and we finally hit our 1miillon mark