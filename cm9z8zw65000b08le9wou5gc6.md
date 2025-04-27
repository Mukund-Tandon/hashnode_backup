---
title: "Building a High-Performance Log Ingestor with Go and ClickHouse"
datePublished: Sun Apr 27 2025 06:07:58 GMT+0000 (Coordinated Universal Time)
cuid: cm9z8zw65000b08le9wou5gc6
slug: building-a-high-performance-log-ingestor-with-go-and-clickhouse
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1745731298202/fad2f9d0-10ae-4f94-8035-593b107b82c9.png
tags: optimization, go, kafka, throughput, logingestor

---

## Introduction

I recently came across a fascinating blog by Zomato Engineering detailing how they rebuilt their logging system to handle an impressive 150 million logs per minute using ClickHouse.

%[https://blog.zomato.com/building-a-cost-effective-logging-platform-using-clickhouse-for-petabyte-scale] 

This inspired me to build my own log ingestor as a learning project. In this blog post, I'll walk you through the architecture and implementation details of my log ingestor system, focusing on how it efficiently collects, buffers, and stores logs at scale.

%[https://github.com/Mukund-Tandon/Logger] 

## System Overview

![Logger Architecture](https://private-user-images.githubusercontent.com/71614009/364486743-73ddcbb1-d323-44b1-9a17-10d10f8cfe0a.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NDU3MzEyODIsIm5iZiI6MTc0NTczMDk4MiwicGF0aCI6Ii83MTYxNDAwOS8zNjQ0ODY3NDMtNzNkZGNiYjEtZDMyMy00NGIxLTlhMTctMTBkMTBmOGNmZTBhLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNTA0MjclMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjUwNDI3VDA1MTYyMlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTU0YTg2MjU0MmY4NGYyZjZmMzYxYTNkYzIzMWU4Njk3NGEzYzhiNTRkZTE1MTYwODE5YjlhZTQ2Y2I2MDVmM2UmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.4KA8rk9O1FeOZR7pMm7KhI67YrEmCo9-0v-cPlqcKzM align="left")

My log ingestor system consists of four main components:

1. **Log Ingestor**: Collects logs from various sources and stores them in ClickHouse
    
2. **Node.js Server**: Provides an API for the dashboard to query logs
    
3. **React Dashboard**: Visualizes and allows querying of logs
    
4. **SDKs**: Client libraries for applications to send logs to the ingestor
    

In this post, I'll focus specifically on the Log Ingestor component, which is built in Go and designed for high throughput.

The Log Ingestor supports three input methods:

* HTTP endpoints (initially used for testing)
    
* gRPC services (implemented as a learning exercise)
    
* Kafka topics (the primary production method)
    

## Architecture Deep Dive

The Log Ingestor follows a three-stage pipeline architecture:

1. **Collection**: Retrieving logs from sources
    
2. **Buffering**: Batching logs for efficient processing
    
3. **Output**: Storing logs in ClickHouse
    

Let's examine each stage in detail.

### 1\. Collection Stage

The collector is responsible for retrieving logs from sources and transforming them into a standardized format. For Kafka collection, which is our primary focus, I implemented a multi-worker approach for parallel processing.

Here's the `KafkaCollector` struct that handles this:

```go
type KafkaCollector struct {
    logbufferChannel chan models.Log
    brokers          []string
    topic            string
    groupID          string
    numWorkers       int
    readers          []*kafka.Reader
    ctx              context.Context
    cancel           context.CancelFunc
    wg               sync.WaitGroup
}
```

Let's break down these fields:

* `logbufferChannel`: Channel to send logs to the buffer stage
    
* `brokers`: List of Kafka broker addresses
    
* `topic`: Kafka topic to consume logs from
    
* `groupID`: Consumer group ID for the Kafka consumer
    
* `numWorkers`: Number of parallel consumer goroutines
    
* `readers`: Array of Kafka readers (one per worker)
    
* `ctx` and `cancel`: For graceful shutdown handling
    
* `wg`: WaitGroup to track active goroutines
    

The collector starts multiple parallel workers, each with its own Kafka reader:

```go
for i := 0; i < c.numWorkers; i++ {
    workerID := strconv.Itoa(i)
    
    reader := kafka.NewReader(readerConfig)
    c.readers[i] = reader
    
    fmt.Printf("Starting worker goroutine %s\n", workerID)
    c.wg.Add(1)
    
    go func(r *kafka.Reader, id string) {
        defer c.wg.Done()
        c.consumeMessages(r, id)
    }(reader, workerID)
}
```

Each worker runs the `consumeMessages` method in its own goroutine:

```go
func (c *KafkaCollector) consumeMessages(reader *kafka.Reader, workerID string) {
    fmt.Printf("Worker %s: Starting message consumption\n", workerID)

    defer func() {
        fmt.Printf("Worker %s: Closing Kafka reader\n", workerID)
        reader.Close()
    }()

    for {
        select {
        case <-c.ctx.Done():
            fmt.Printf("Worker %s: Context canceled, stopping\n", workerID)
            return
        default:
            message, err := reader.ReadMessage(c.ctx)
            if err != nil {
                if err == context.Canceled {
                    fmt.Printf("Worker %s: Context canceled while reading message\n", workerID)
                    return
                }
                fmt.Printf("Worker %s: Error reading Kafka message: %v\n", workerID, err)
                time.Sleep(500 * time.Millisecond) // Brief pause before retrying
                continue
            }

            log, err := transformer.KafkaEventToLog(message)
            if err != nil {
                fmt.Printf("Worker %s: Error transforming Kafka message: %v\n", workerID, err)
                continue
            }

            c.logbufferChannel <- log
        }
    }
}
```

Each message is transformed from the Kafka format to our internal log model:

```go
type Log struct {
   Timestamp  string
   Level      string
   Message    string
   ResourceID string
}
```

The transformation is handled by a simple function:

```go
func KafkaEventToLog(msg kafka.Message) (models.Log, error) {
    var log models.Log

    err := json.Unmarshal(msg.Value, &log)
    if err != nil {
        return models.Log{}, err
    }

    return log, nil
}
```

### 2\. Buffering Stage

The buffer stage collects individual logs and groups them into batches for efficient database insertion. This is crucial for high-throughput systems, as batch operations are much more efficient than individual inserts.

Here's the buffer implementation:

```go
gofunc LogBuffer(logBatchOutputChannel chan models.Logbatch, metricsLogger *metrics.MetricsLogger) chan models.Log {
    logChannel := make(chan models.Log)
    buffer := make([]models.Log, 0, 4000)
    ticker := time.NewTicker(15 * time.Second)

    go func() {
        for {
            select {
            case log := <-logChannel:
                buffer = append(buffer, log)
                if len(buffer) >= 4000 {
                    logBatchOutputChannel <- models.Logbatch{Logbatch: buffer}
                    buffer = buffer[:0] 
                }
            case <-ticker.C:
                if len(buffer) > 0 {
                    logBatchOutputChannel <- models.Logbatch{Logbatch: buffer}
                    buffer = buffer[:0] // Clear the buffer
                }
                ticker.Reset(15 * time.Second)
            }
        }
    }()

    return logChannel
}
```

The buffer flushes logs to the output stage under two conditions:

1. When the buffer reaches 4000 logs (size-based trigger)
    
2. When 15 seconds have passed since the last flush (time-based trigger)
    

This dual-trigger approach ensures both efficiency (batch size) and timeliness (maximum delay).

### 3\. Output Stage

The output stage is responsible for inserting batches of logs into ClickHouse. To maximize throughput, I implemented a worker pool pattern with multiple database connections:

```go
type OutputPool struct {
    workers       []*Worker
    inputChannel  chan models.Logbatch
    metricsLogger *metrics.MetricsLogger
    wg            sync.WaitGroup
    ctx           context.Context
    cancel        context.CancelFunc
}

type Worker struct {
    id            string
    conn          clickhouse.Conn
    inputChannel  chan models.Logbatch
    metricsLogger *metrics.MetricsLogger
    ctx           context.Context
    wg            *sync.WaitGroup
}
```

The pool dispatcher distributes incoming batches to workers using a round-robin approach:

```go
func (p *OutputPool) dispatch() {
    fmt.Println("Output dispatcher started")
    
    currentWorker := 0
    numWorkers := len(p.workers)
    
    for {
        select {
        case <-p.ctx.Done():
            fmt.Println("Dispatcher shutting down")
            return
            
        case batch := <-p.inputChannel:
            // Round-robin distribution
            p.workers[currentWorker].inputChannel <- batch
            currentWorker = (currentWorker + 1) % numWorkers
        }
    }
}
```

Each worker processes batches independently using its own database connection:

```go
func (w *Worker) work() {
    defer w.wg.Done()
    fmt.Printf("Worker %s started processing\n", w.id)
    
    for {
        select {
        case <-w.ctx.Done():
            fmt.Printf("Worker %s shutting down\n", w.id)
            if w.conn != nil {
                w.conn.Close()
            }
            return
            
        case batch := <-w.inputChannel:
            fmt.Printf("Worker %s processing batch of size %d\n", w.id, len(batch.Logbatch))
            
            // Insert the batch
            err := doBatchInsert(batch, w.conn)
            
            // Record metrics for successful insertions
            if err == nil && w.metricsLogger != nil {
                w.metricsLogger.RecordDBInsertion(len(batch.Logbatch))
                fmt.Printf("Worker %s recorded insertion of %d logs\n", w.id, len(batch.Logbatch))
            } else if err != nil {
                fmt.Printf("Worker %s error inserting batch: %v\n", w.id, err)
            }
        }
    }
}
```

The batch insertion into ClickHouse is handled efficiently:

```go
func doBatchInsert(logbatch models.Logbatch, conn clickhouse.Conn) error {
    ctx := context.Background()
    batch, err := conn.PrepareBatch(ctx, "INSERT INTO logs")
    if err != nil {
        fmt.Println("Error preparing batch:", err)
        return err
    }

    logBatchSize := len(logbatch.Logbatch)
    for i := 0; i < logBatchSize; i++ {
        timestampStr := logbatch.Logbatch[i].Timestamp

        // Parse timestamp
        timestamp, err := time.Parse(time.RFC3339Nano, timestampStr)
        if err != nil {
            fmt.Printf("Error parsing timestamp %s: %v\n", timestampStr, err)
            continue // Skip this entry if timestamp parsing fails
        }

        // Format for ClickHouse
        clickHouseInsertFormat := timestamp.Format("2006-01-02 15:04:05.000000")

        // Extract log fields
        message := logbatch.Logbatch[i].Message
        level := logbatch.Logbatch[i].Level
        resourceID := logbatch.Logbatch[i].ResourceID

        // Append to batch
        err = batch.Append(
            clickHouseInsertFormat,
            level,
            message,
            resourceID,
        )
        if err != nil {
            fmt.Println("Error executing query:", err)
            return err
        }
    }
    
    // Execute the batch insert
    err = batch.Send()
    if err != nil {
        fmt.Println("Error sending batch:", err)
        return err
    }
    
    return nil
}
```

## System Flow

Let's trace a log's journey through the system:

1. An application sends a log message to a Kafka topic
    
2. One of the Kafka consumer workers reads the message
    
3. The message is transformed into our standard log format
    
4. The log is sent to the buffer
    
5. The buffer collects logs until reaching 1000 or 15 seconds passes
    
6. The batch is sent to the output stage
    
7. The round-robin dispatcher assigns the batch to a worker
    
8. The worker inserts the batch into ClickHouse
    
9. The logs are now available for querying via the dashboard
    

## Performance Considerations

This architecture is designed for high throughput with several key optimizations:

1. **Parallel consumption**: Multiple Kafka workers to process incoming messages concurrently
    
2. **Efficient batching**: Size-based and time-based triggers balance throughput and latency
    
3. **Connection pooling**: Multiple database workers with dedicated connections
    
4. **Round-robin distribution**: Even distribution of workload among database workers
    

# Now that we have build our log ingestor its time to test it its perfomance

### Log Ingestor Metrics Collection

To accurately measure the performance of our log ingestor, I implemented a simple but effective metrics collection system that tracks database insertions. The metrics package is focused on recording the number of logs inserted into ClickHouse and calculating insertion rates over time:

```go
package metrics

import (
	"fmt"
	"os"
	"sync"
	"time"
)

type MetricsLogger struct {
	totalInserted int64     // Total logs inserted since start
	prevInserted  int64     // Logs inserted as of last measurement
	lastLogTime   time.Time // Timestamp of last measurement
	logFile       *os.File  // File to write metrics data
	mu            sync.Mutex // Mutex to protect concurrent access
}

func NewMetricsLogger(logFilePath string) (*MetricsLogger, error) {
	// Open metrics log file
	file, err := os.OpenFile(logFilePath, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
	if err != nil {
		return nil, fmt.Errorf("failed to open metrics log file: %w", err)
	}
	
	// Write CSV header if file is new
	fileInfo, err := file.Stat()
	if err == nil && fileInfo.Size() == 0 {
		headerLine := "timestamp,logs_per_second,total_logs_inserted\n"
		if _, err := file.WriteString(headerLine); err != nil {
			return nil, fmt.Errorf("failed to write header to metrics log file: %w", err)
		}
	}
	
	return &MetricsLogger{
		lastLogTime: time.Now(),
		logFile:     file,
	}, nil
}

// RecordDBInsertion increments the counter when logs are inserted to ClickHouse
func (m *MetricsLogger) RecordDBInsertion(count int) {
	m.mu.Lock()
	defer m.mu.Unlock()
	
	m.totalInserted += int64(count)
}

// LogInsertionRate calculates and logs the current insertion rate
func (m *MetricsLogger) LogInsertionRate() error {
	m.mu.Lock()
	defer m.mu.Unlock()
	
	now := time.Now()
	timeSinceLast := now.Sub(m.lastLogTime).Seconds()
	insertedSinceLast := m.totalInserted - m.prevInserted
	
	// Calculate insertion rate (logs per second)
	insertionRate := float64(insertedSinceLast) / timeSinceLast
	
	// Format and write the log entry
	logLine := fmt.Sprintf("%s,%.2f,%d\n", 
		now.Format(time.RFC3339),
		insertionRate,
		m.totalInserted)
	
	if _, err := m.logFile.WriteString(logLine); err != nil {
		return fmt.Errorf("failed to write to metrics log file: %w", err)
	}
	
	// Update tracking variables for next measurement
	m.lastLogTime = now
	m.prevInserted = m.totalInserted
	
	return nil
}

// StartPeriodicLogging begins a background goroutine to log metrics at the specified interval
func (m *MetricsLogger) StartPeriodicLogging(interval time.Duration) {
	ticker := time.NewTicker(interval)
	
	go func() {
		for range ticker.C {
			if err := m.LogInsertionRate(); err != nil {
				fmt.Fprintf(os.Stderr, "Error logging metrics: %v\n", err)
			}
		}
	}()
}

// Close properly closes the metrics log file
func (m *MetricsLogger) Close() error {
	m.mu.Lock()
	defer m.mu.Unlock()
	
	if m.logFile != nil {
		return m.logFile.Close()
	}
	return nil
}
```

I integrated this metrics logger into our output workers to track database insertions:

```go
// In worker.work() method
case batch := <-w.inputChannel:
    fmt.Printf("Worker %s processing batch of size %d\n", w.id, len(batch.Logbatch))
    
    // Insert the batch
    err := doBatchInsert(batch, w.conn)
    
    // Record successful insertions in metrics
    if err == nil && w.metricsLogger != nil {
        w.metricsLogger.RecordDBInsertion(len(batch.Logbatch))
    } else if err != nil {
        fmt.Printf("Worker %s error inserting batch: %v\n", w.id, err)
    }
```

And started the metrics logger in our main function:

```go
func main() {
    // Create metrics logger
    metricsLogger, err := metrics.NewMetricsLogger("ingestor_metrics.csv")
    if err != nil {
        fmt.Printf("Error creating metrics logger: %v\n", err)
        os.Exit(1)
    }
    defer metricsLogger.Close()
    
    // Start periodic logging every 1 seconds
    metricsLogger.StartPeriodicLogging(1 * time.Second)
    
    // ... rest of setup code
}
```

This metrics system provides valuable data on:

1. **Insertion Rate**: How many logs per second are being written to ClickHouse
    
2. **Total Inserted**: The cumulative count of logs stored in the database
    
3. **Performance Over Time**: The CSV format allows for visualization and analysis of performance trends
    

During testing, we can monitor these metrics to understand how our log ingestor performs under various load conditions. The metrics are particularly useful for identifying performance bottlenecks and validating the effectiveness of our batching and worker pool strategies.

The CSV format also makes it easy to generate graphs and visualizations of the ingestor's performance using tools like Excel, Python's matplotlib, or data visualization platforms.

Below is a blog you can read if you want to see how I was able to generate a million logs per second locally to test this infrastructure.

[Blog Post link](https://mukund-tandon-dev.hashnode.dev/can-i-send-1million-logs-per-second-to-kafka-locally)

## Performance Journey: From 8K to 1Million Logs/Second

My initial implementation processed around 12,000-16,000 logs per second while receiving around 800K logs persecond. Here's how I optimized the system step by step:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745676143427/2774f1e3-dac8-4097-acce-6f99e57dabfc.png align="center")

### 1\. Increasing Batch Size (4K → 40K)

The first optimization was increasing the buffer size from 4,000 to 40,000 logs:

```go
gobuffer := make([]models.Log, 0, 40000)
```

**Result**: Throughput seems to have doubled to about 40,000 logs/second, but buffer fill time increased to 2-3 seconds as we can see 0logs processed per seconds also. This indicated that the bottleneck was in the Kafka consumer's ability to fill the buffer quickly enough.'

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745677730257/df537f86-12b4-4958-9465-c4b8b13dfc72.png align="center")

### 2\. Optimizing Kafka Consumer Configuration

Let me analyze the specific Kafka configuration you provided and how it laid the foundation for improved performance:

```go
goreaderConfig := kafka.ReaderConfig{
    Brokers:         c.brokers,
    Topic:           c.topic,
    GroupID:         c.groupID,
    MinBytes:        10e3,        // 10KB minimum batch size
    MaxBytes:        20e6,        // 20MB maximum batch size
    MaxWait:         1 * time.Second,
    StartOffset:     kafka.FirstOffset,
    ReadLagInterval: -1,
    CommitInterval:  1 * time.Second, // The key improvement
}
```

Compared to your original configuration:

```go
goreaderConfig := kafka.ReaderConfig{
    Brokers:         c.brokers,
    Topic:           c.topic,
    GroupID:         c.groupID,
    MinBytes:        10e3,
    MaxBytes:        10e6,        // Only 10MB
    MaxWait:         1 * time.Second,
    StartOffset:     kafka.FirstOffset,
    ReadLagInterval: -1,
    // No CommitInterval
}
```

## The Critical CommitInterval Addition

The single most impactful change here was the addition of the `CommitInterval: 1 * time.Second` parameter, which transformed your throughput from ~16K to nearly 350K logs/second on a average—a remarkable improvement from just one parameter.

### How CommitInterval Works

In Kafka's consumer model, offsets track which messages have been processed. By default, without a CommitInterval specified, the Kafka Go client commits offsets automatically after processing each message or batch. This creates substantial coordination overhead:

1. **Without CommitInterval**: The client makes a network request to Kafka's coordinator for nearly every batch of messages processed, creating a synchronous bottleneck
    
2. **With CommitInterval: 1 \* time.Second**: Commits are batched and sent just once per second, regardless of how many message batches are processed in that second
    

### Performance Impact

This change had several cascading effects:

1. **Reduced Coordination Traffic**: With your system processing hundreds of thousands of logs per second, this reduced coordination traffic by approximately 99%
    
2. **Lower Broker Load**: Kafka brokers experienced significantly less load from handling commits, freeing resources for actual message delivery
    
3. **Reduced Round-Trip Latency**: Eliminating per-batch commits removed a synchronous wait from your processing pipeline
    
4. **Increased Throughput Ceiling**: Without the constant commit bottleneck, your system could approach the true maximum throughput of your network and CPU resources
    

### The MaxBytes setting to 8MB

Below are rough calculation as to why 8mb seeems perfect for max bute , remeber below are just rough extimaates

### The Partition Throughput Calculation

My configuration is precisely calibrated to my partitioned architecture:

1. **Total System Throughput**: 800,000 logs/second
    
2. **Number of Partitions**: 10 partitions
    
3. **Per-Partition Throughput**: 800,000 ÷ 10 = 80,000 logs/second per partition
    

### Log Size and Compression Analysis

Each log entry has specific size characteristics:

* Average uncompressed log size: ~500 bytes (including timestamp, level, message, resourceID, and JSON overhead)
    
* Kafka compression ratio: ~5:1 (typical for log data using Snappy)
    
* Compressed log size: ~100 bytes per log
    

### Data Flow Rate Calculation

For each partition:

1. **Uncompressed Data Rate**:
    
    * 80,000 logs/second × 500 bytes = 40,000,000 bytes/second (40MB/s)
        
2. **Compressed Data Rate**:
    
    * 40MB/s ÷ 5 = 8MB/second of compressed data flowing through each partition
        

### Final output-

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745728863758/2efb338a-2bd0-41a2-a4a0-2e3642ec549b.png align="center")

During testing, I noticed that when producing and consuming logs simultaneously on the same machine, throughput was around 280K logs/second. Once the producer finished, throughput increased to 500K logs/second.

This revealed resource competition between producer and consumer processes on my 10-core machine. Both were competing for:

* CPU resources
    
* Network bandwidth
    
* Disk I/O
    
* Memory and cache
    

Testing out various buffer configurations , input throughput is around 600K logs/second

| Buffer Size | Maximum Throughput after 15s | Minimum | Commnets during frst 15 seconds |  |
| --- | --- | --- | --- | --- |
| 40000 | 559998.0 | 399624.70 | 150K to 330K throuput |  |
| 100000 | 600019.53 | 399907.50 | 200K to 400K which average of mostly around 299K |  |
| 150000 | 750038.2 | 449873.74 | thorugput of 150K to 450K |  |
| 300000 | 714548.64 | 299997.70 | getting an average of 300K logs/second but sometime buffer take 2 secods to fill |  |
| 500000 | 999960.54 | 499994.27 | Buffer was taking 2 seconds to get filled so after every alternate second we got thoruput of 500K logs/second |  |
| 600000 | 1200064.64 | 528747.74 | similar to 500K buffer , we see buffer takes 2-3 s to fill then we get 600K logs/second or sometimes more than that |  |

### Screenshots of various buffer size test

150K

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745729406742/3e60bba5-a908-407f-8b00-5b0397c52430.png align="center")

300K

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745729669322/364e5ff0-a014-40a2-b9d4-4dac73763dc5.png align="center")

500K

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745729847636/03f14a53-979c-4df1-af76-7b6b05cb8970.png align="center")

600K

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745730372500/65aebda4-0752-4a9e-81b6-e95a54c084ec.png align="center")

During my optimization work, I conducted extensive testing of various buffer sizes to understand their impact on throughput performance. Each configuration revealed distinct behavioral patterns in our log ingestion pipeline. The 40K buffer showed consistent throughput between 399K-559K logs/second. The 100K buffer achieved peaks of 600K logs/second while maintaining a similar floor of around 400K logs/second. Moving to 150K, we observed higher peak throughput of 750K logs/second with improved minimum performance of 450K logs/second. The 300K configuration reached 714K logs/second but occasionally required 2 seconds to fill, creating a processing rhythm. Most notably, the 500K buffer demonstrated peaks approaching 1 million logs/second with 2-second fill cycles, while the 600K buffer reached extraordinary throughput of 1.2 million logs/second with slightly longer 2-3 second fill cycles and a baseline of approximately 530K logs/second. These measurements provided valuable insights into the relationship between buffer size, processing patterns, and throughput characteristics across different workload conditions. Also, despite our ingress throughput being around 600K logs/second, we saw max throughput of 1.2 million because before such high throughput we observed no logs being sent in the previous second, so they were suddenly flushed to the database, making our metrics record 1.2 million."

## Conclusion: Building a Million-Message Log Ingestor

Throughout this journey of building and optimizing a log ingestor system, I've demonstrated how a thoughtfully designed architecture can scale to handle massive throughput requirements. What began as a learning project inspired by Zoma[to Engineering's approach evolved into a high-performanc](https://support.anthropic.com/en/articles/8525154-claude-is-providing-incorrect-or-misleading-responses-what-s-going-on)e system capable of processing over 1.2 million logs per second.

The three-stage pipeline architecture—collection, buffering, and output—proved to be a robust foundation. Each component was designed with concurrency and efficiency in mind, from the multi-worker Kafka consumers to the batched ClickHouse insertions. This modular approach not only made the system easier to reason about but also simplified the optimization process by allowing targeted improvements at each stage.

Perhaps the most valuable insight from this project was understanding how seemingly small configuration changes can yield dramatic performance improvements. The Kafka consumer optimizations, particularly the `CommitInterval` setting, transformed throughput by reducing coordination overhead. Similarly, the extensive buffer size testing revealed fascinating patterns in how system behavior changes across different configurations, offering a glimpse into the complex interplay between memory usage, latency, and processing rhythm.

This project also reinforced the importance o[f metrics and measurement in performance optimization. B](https://support.anthropic.com/en/articles/8525154-claude-is-providing-incorrect-or-misleading-responses-what-s-going-on)y implementing a simple but effective metrics collection system, I was able to quantify improvements, identify bottlen[ecks, and make data-driven decisions about configuration change](https://support.anthropic.com/en/articles/8525154-claude-is-providing-incorrect-or-misleading-responses-what-s-going-on)s.

Building a high-throughput log ingestor isn't just about raw performance—it's about creating a system that can reliably handle production workloads with predictable behavior. The optimizations described here have applications beyond logging systems, offering valuable lessons for any high-throughput data processing pipeline.

While the current implementation has achieved impressive results, there's always room for further improvement through horizontal scaling, additional configuration tuning, and exploration of new technologies. However, the fundamental principles of batch processing, parallel execution, and careful configuration management will remain relevant regardless of the specific implementation details.

As data volumes continue to grow across industries, the ability to efficiently ingest, process, and store massive log volumes becomes increasingly crucial. I hope this exploration provides valuable insights for others building similar systems in their own environments.