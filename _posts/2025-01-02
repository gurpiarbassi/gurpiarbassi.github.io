---
layout: post
title: JVM Tuning
published: true
categories: 
            - JVM

---

# JVM Garbage Collection

## Garbage Collector Types

### Serial GC (-XX:+UseSerialGC)
- Single-threaded collector
- Simple and efficient for small heaps
- Best for:
  - Single CPU environments
  - Small heaps (<4GB)
  - Batch processing
- Set to "ergonomic" by default, meaning JVM chooses based on system resources

### Parallel GC (-XX:+UseParallelGC)
- Multi-threaded collector
- Focuses on throughput
- Best for:
  - Batch processing
  - Scientific computing
  - When throughput is more important than latency
- Tuning parameters:
  - `-XX:ParallelGCThreads=n`: Number of GC threads
  - `-XX:MaxGCPauseMillis=n`: Target pause time

### G1GC (-XX:+UseG1GC)
- Garbage-First Garbage Collector
- Region-based collector
- Best for:
  - Large heaps (>4GB)
  - Low latency requirements
  - Mixed workloads
- Tuning parameters:
  - `-XX:MaxGCPauseMillis=n`: Target pause time
  - `-XX:G1HeapRegionSize=n`: Region size

### ZGC (-XX:+UseZGC)
- Low-latency collector
- Concurrent processing
- Best for:
  - Very large heaps
  - Sub-millisecond pause times
  - Response-time critical applications

## GC Tuning Best Practices

### Memory Settings
- `-Xms`: Initial heap size
  - Set equal to -Xmx for predictable performance
- `-Xmx`: Maximum heap size
  - MB format: `356m`
  - GB format: `2g`

### Container Settings
- `-XX:MaxRAMPercentage`: Heap size as percentage of available RAM
  - Example: `-XX:MaxRAMPercentage=75.0`
- `-XX:InitialRAMPercentage`: Initial heap size percentage
- `-XX:MinRAMPercentage`: Minimum heap size percentage

### Monitoring
- `-XX:+PrintGCDetails`: Print detailed GC information
- `-XX:+PrintGCDateStamps`: Add timestamps to GC logs
- `-Xlog:gc*`: Unified logging for GC (Java 11+)

### Common Issues
1. Long GC pauses
   - Reduce heap size
   - Consider switching to G1GC or ZGC
2. High CPU usage
   - Check number of GC threads
   - Monitor GC frequency
3. Memory leaks
   - Use heap dumps
   - Monitor old generation growth

## Default Garbage Collector
The statement "G1GC is the default garbage collector in the JVM" is not entirely accurate. Java applies specific rules to determine if G1GC should be used by default. It really depends on how much memory is available.

## Checking Current GC
To see what garbage collector your application is using you can use the `PrintFlagsFinal` JVM option. For example here were are using it in the `java -version` command:

```bash
java -XX:+PrintFlagsFinal -version | grep -e "Use.*GC"
```

Example output:
```
bool UseAdaptiveSizeDecayMajorGCCost          = true    {product} {default}
bool UseAdaptiveSizePolicyWithSystemGC        = false   {product} {default}
bool UseDynamicNumberOfGCThreads              = true    {product} {default}
bool UseG1GC                                  = true    {product} {default}
bool UseGCOverheadLimit                       = true    {product} {default}
bool UseMaximumCompactionOnSystemGC           = true    {product} {default}
bool UseParallelGC                            = false   {product} {default}
bool UseSerialGC                              = false   {product} {ergonomic}
bool UseShenandoahGC                          = false   {product} {default}
bool UseZGC                                   = false   {product} {default}
```

Note: Serial GC is set to "ergonomic", meaning the JVM will use it if appropriate for the system.

## Best Practices

### ThreadPoolExecutor Configuration
Always configure ThreadPoolExecutor appropriately.

### Container-Friendly Settings
- `-XX:MaxRAMPercentage`: Set as percentage
  - Great for workloads that need to scale with container memory limits
