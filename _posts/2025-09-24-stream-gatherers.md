---
layout: post
title: "Java Stream Gatherers API: Custom Intermediate Operations"
published: true
date: 2025-09-24
categories: java programming streams
---

# Motivation

In this post, we'll explore Java's **Stream Gatherers API** - a powerful feature introduced in Java 24 that allows you to create custom intermediate operations for streams. This API fills a significant gap in the Stream API by enabling stateful transformations and complex data processing patterns directly within stream pipelines.

## The Problem: Limited Stream Operations

### Traditional Stream API Limitations

```java
// What we can do with standard Stream API
List<String> result = Stream.of("apple", "banana", "cherry")
    .filter(s -> s.length() > 5)
    .map(String::toUpperCase)
    .sorted()
    .toList();

// What we CAN'T do easily - complex stateful operations
// - Windowing (sliding windows)
// - Batching with custom logic
// - Stateful transformations
// - Complex grouping patterns
```

### Workarounds Before Gatherers

```java
// Old way - using collectors (terminal operation)
List<List<String>> batches = Stream.of("a", "b", "c", "d", "e")
    .collect(Collectors.groupingBy(
        s -> Stream.of("a", "b", "c", "d", "e").toList().indexOf(s) / 2
    ))
    .values()
    .stream()
    .map(List::copyOf)
    .toList();

// Old way - external state management
List<String> windowed = new ArrayList<>();
List<String> buffer = new ArrayList<>();
for (String item : items) {
    buffer.add(item);
    if (buffer.size() == 3) {
        windowed.add(String.join("-", buffer));
        buffer.remove(0); // sliding window
    }
}
```

## What are Stream Gatherers?

**Stream Gatherers** are custom intermediate operations that can maintain state and perform complex transformations on stream elements. They bridge the gap between simple stateless operations (like `map`, `filter`) and terminal operations (like `collect`).

### Key Characteristics

- **Stateful**: Can maintain internal state across elements
- **Intermediate**: Can be chained with other stream operations
- **Customizable**: Define your own transformation logic
- **Parallel-safe**: Support parallel stream processing
- **Composable**: Can be combined with other gatherers

## The Gatherer Interface

```java
public interface Gatherer<T, A, R> {
    Supplier<A> initializer();
    Integrator<A, T, R> integrator();
    BinaryOperator<A> combiner();        // Optional - for parallel processing
    BiConsumer<A, Downstream<? super R>> finisher(); // Optional - for cleanup
}
```

### Interface Components

- **T**: Input element type
- **A**: Accumulator/state type
- **R**: Output element type
- **initializer()**: Creates initial state
- **integrator()**: Processes each element
- **combiner()**: Merges states for parallel processing
- **finisher()**: Final cleanup and output

## Built-in Gatherers

Java provides several useful built-in gatherers in `java.util.stream.Gatherers`. Let's explore each one with "before" and "after" examples to see how they simplify complex operations.

### 1. **Windowing Operations**

#### Fixed-Size Windows

**Before (Manual Implementation):**
```java
// Complex manual implementation
public static <T> List<List<T>> createFixedWindows(List<T> list, int windowSize) {
    List<List<T>> windows = new ArrayList<>();
    List<T> currentWindow = new ArrayList<>();
    
    for (T element : list) {
        currentWindow.add(element);
        if (currentWindow.size() == windowSize) {
            windows.add(new ArrayList<>(currentWindow));
            currentWindow.clear();
        }
    }
    
    // Handle remaining elements
    if (!currentWindow.isEmpty()) {
        windows.add(new ArrayList<>(currentWindow));
    }
    
    return windows;
}

List<String> data = Arrays.asList("a", "b", "c", "d", "e", "f");
List<List<String>> fixedWindows = createFixedWindows(data, 3);
// Result: [[a, b, c], [d, e, f]]
```

**After (With Gatherers):**
```java
// Simple and clean with gatherers
List<List<String>> fixedWindows = Stream.of("a", "b", "c", "d", "e", "f")
    .gather(Gatherers.windowFixed(3))
    .toList();
// Result: [[a, b, c], [d, e, f]]
```

#### Sliding Windows

**Before (Manual Implementation):**
```java
// Complex sliding window implementation
public static <T> List<List<T>> createSlidingWindows(List<T> list, int windowSize) {
    List<List<T>> windows = new ArrayList<>();
    
    for (int i = 0; i <= list.size() - windowSize; i++) {
        List<T> window = new ArrayList<>();
        for (int j = i; j < i + windowSize; j++) {
            window.add(list.get(j));
        }
        windows.add(window);
    }
    
    return windows;
}

List<String> data = Arrays.asList("a", "b", "c", "d", "e");
List<List<String>> slidingWindows = createSlidingWindows(data, 3);
// Result: [[a, b, c], [b, c, d], [c, d, e]]
```

**After (With Gatherers):**
```java
// Elegant sliding windows
List<List<String>> slidingWindows = Stream.of("a", "b", "c", "d", "e")
    .gather(Gatherers.windowSliding(3))
    .toList();
// Result: [[a, b, c], [b, c, d], [c, d, e]]
```

### 2. **Scan Operation (Cumulative Operations)**

#### Cumulative Sum

**Before (Manual Implementation):**
```java
// Manual cumulative sum
public static List<Integer> calculateCumulativeSum(List<Integer> numbers) {
    List<Integer> result = new ArrayList<>();
    int sum = 0;
    
    for (Integer number : numbers) {
        sum += number;
        result.add(sum);
    }
    
    return result;
}

List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
List<Integer> cumulativeSum = calculateCumulativeSum(numbers);
// Result: [1, 3, 6, 10, 15]
```

**After (With Gatherers):**
```java
// Clean cumulative sum with scan
List<Integer> cumulativeSum = Stream.of(1, 2, 3, 4, 5)
    .gather(Gatherers.scan(0, Integer::sum))
    .toList();
// Result: [1, 3, 6, 10, 15]
```

#### Running Maximum

**Before (Manual Implementation):**
```java
// Manual running maximum
public static List<Integer> calculateRunningMax(List<Integer> numbers) {
    List<Integer> result = new ArrayList<>();
    int max = Integer.MIN_VALUE;
    
    for (Integer number : numbers) {
        max = Math.max(max, number);
        result.add(max);
    }
    
    return result;
}

List<Integer> numbers = Arrays.asList(3, 1, 4, 1, 5, 9, 2);
List<Integer> runningMax = calculateRunningMax(numbers);
// Result: [3, 3, 4, 4, 5, 9, 9]
```

**After (With Gatherers):**
```java
// Elegant running maximum
List<Integer> runningMax = Stream.of(3, 1, 4, 1, 5, 9, 2)
    .gather(Gatherers.scan(Integer.MIN_VALUE, Integer::max))
    .toList();
// Result: [3, 3, 4, 4, 5, 9, 9]
```

#### Running Average

**Before (Manual Implementation):**
```java
// Complex running average calculation
public static List<Double> calculateRunningAverage(List<Integer> numbers) {
    List<Double> result = new ArrayList<>();
    int sum = 0;
    int count = 0;
    
    for (Integer number : numbers) {
        sum += number;
        count++;
        result.add((double) sum / count);
    }
    
    return result;
}

List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
List<Double> runningAvg = calculateRunningAverage(numbers);
// Result: [1.0, 1.5, 2.0, 2.5, 3.0]
```

**After (With Gatherers):**
```java
// Clean running average with custom accumulator
class RunningAverage {
    private int sum = 0;
    private int count = 0;
    
    public double add(int value) {
        sum += value;
        count++;
        return (double) sum / count;
    }
}

List<Double> runningAvg = Stream.of(1, 2, 3, 4, 5)
    .gather(Gatherers.scan(new RunningAverage(), RunningAverage::add))
    .toList();
// Result: [1.0, 1.5, 2.0, 2.5, 3.0]
```

### 3. **Fold Operation (Reduction)**

#### String Concatenation

**Before (Manual Implementation):**
```java
// Manual string concatenation
public static String concatenateStrings(List<String> strings) {
    StringBuilder result = new StringBuilder();
    
    for (String str : strings) {
        result.append(str);
    }
    
    return result.toString();
}

List<String> words = Arrays.asList("Hello", " ", "World", "!");
String concatenated = concatenateStrings(words);
// Result: "Hello World!"
```

**After (With Gatherers):**
```java
// Simple fold operation
String concatenated = Stream.of("Hello", " ", "World", "!")
    .gather(Gatherers.fold("", String::concat))
    .findFirst()
    .orElse("");
// Result: "Hello World!"
```

#### Product Calculation

**Before (Manual Implementation):**
```java
// Manual product calculation
public static int calculateProduct(List<Integer> numbers) {
    int product = 1;
    
    for (Integer number : numbers) {
        product *= number;
    }
    
    return product;
}

List<Integer> numbers = Arrays.asList(2, 3, 4, 5);
int product = calculateProduct(numbers);
// Result: 120
```

**After (With Gatherers):**
```java
// Clean product calculation
int product = Stream.of(2, 3, 4, 5)
    .gather(Gatherers.fold(1, (a, b) -> a * b))
    .findFirst()
    .orElse(1);
// Result: 120
```

### 4. **Map Concurrent (Parallel Processing)**

#### File Processing

**Before (Manual Implementation):**
```java
// Complex concurrent processing
public static List<String> processFilesConcurrently(List<String> filenames, int maxConcurrency) {
    ExecutorService executor = Executors.newFixedThreadPool(maxConcurrency);
    List<CompletableFuture<String>> futures = new ArrayList<>();
    
    for (String filename : filenames) {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> 
            processFile(filename), executor);
        futures.add(future);
    }
    
    List<String> results = futures.stream()
        .map(CompletableFuture::join)
        .toList();
    
    executor.shutdown();
    return results;
}

private String processFile(String filename) {
    // Simulate file processing
    try {
        Thread.sleep(100); // Simulate I/O
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    return "Processed: " + filename;
}

List<String> filenames = Arrays.asList("file1.txt", "file2.txt", "file3.txt", "file4.txt");
List<String> processed = processFilesConcurrently(filenames, 3);
// Result: ["Processed: file1.txt", "Processed: file2.txt", ...]
```

**After (With Gatherers):**
```java
// Simple concurrent processing
List<String> processed = Stream.of("file1.txt", "file2.txt", "file3.txt", "file4.txt")
    .gather(Gatherers.mapConcurrent(3, this::processFile))
    .toList();

private String processFile(String filename) {
    // Simulate file processing
    try {
        Thread.sleep(100); // Simulate I/O
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    return "Processed: " + filename;
}
// Result: ["Processed: file1.txt", "Processed: file2.txt", ...]
```

#### API Calls with Rate Limiting

**Before (Manual Implementation):**
```java
// Complex rate-limited API calls
public static List<String> callApisWithRateLimit(List<String> endpoints, int maxConcurrency) {
    ExecutorService executor = Executors.newFixedThreadPool(maxConcurrency);
    Semaphore semaphore = new Semaphore(maxConcurrency);
    List<CompletableFuture<String>> futures = new ArrayList<>();
    
    for (String endpoint : endpoints) {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            try {
                semaphore.acquire();
                return callApi(endpoint);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return "Error: " + endpoint;
            } finally {
                semaphore.release();
            }
        }, executor);
        futures.add(future);
    }
    
    List<String> results = futures.stream()
        .map(CompletableFuture::join)
        .toList();
    
    executor.shutdown();
    return results;
}

private String callApi(String endpoint) {
    // Simulate API call
    return "Response from: " + endpoint;
}

List<String> endpoints = Arrays.asList("/api/users", "/api/orders", "/api/products");
List<String> responses = callApisWithRateLimit(endpoints, 2);
```

**After (With Gatherers):**
```java
// Clean rate-limited API calls
List<String> responses = Stream.of("/api/users", "/api/orders", "/api/products")
    .gather(Gatherers.mapConcurrent(2, this::callApi))
    .toList();

private String callApi(String endpoint) {
    // Simulate API call
    return "Response from: " + endpoint;
}
```

### 5. **Additional Built-in Gatherers**

#### Take While (Conditional Processing)

**Before (Manual Implementation):**
```java
// Manual take while implementation
public static <T> List<T> takeWhile(List<T> list, Predicate<T> predicate) {
    List<T> result = new ArrayList<>();
    
    for (T element : list) {
        if (predicate.test(element)) {
            result.add(element);
        } else {
            break;
        }
    }
    
    return result;
}

List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9);
List<Integer> taken = takeWhile(numbers, n -> n < 5);
// Result: [1, 2, 3, 4]
```

**After (With Gatherers):**
```java
// Simple take while
List<Integer> taken = Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)
    .gather(Gatherers.takeWhile(n -> n < 5))
    .toList();
// Result: [1, 2, 3, 4]
```

#### Drop While (Skip Until Condition)

**Before (Manual Implementation):**
```java
// Manual drop while implementation
public static <T> List<T> dropWhile(List<T> list, Predicate<T> predicate) {
    List<T> result = new ArrayList<>();
    boolean dropping = true;
    
    for (T element : list) {
        if (dropping && predicate.test(element)) {
            continue; // Skip this element
        } else {
            dropping = false;
            result.add(element);
        }
    }
    
    return result;
}

List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9);
List<Integer> dropped = dropWhile(numbers, n -> n < 5);
// Result: [5, 6, 7, 8, 9]
```

**After (With Gatherers):**
```java
// Simple drop while
List<Integer> dropped = Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)
    .gather(Gatherers.dropWhile(n -> n < 5))
    .toList();
// Result: [5, 6, 7, 8, 9]
```

## Creating Custom Gatherers

### 1. **Simple Stateless Gatherer**

```java
// Custom gatherer to add prefixes
static Gatherer<String, ?, String> addPrefix(String prefix) {
    return Gatherer.of(
        (_, element, downstream) -> downstream.push(prefix + element)
    );
}

// Usage
List<String> prefixed = Stream.of("apple", "banana")
    .gather(addPrefix("fruit: "))
    .toList();
// Result: ["fruit: apple", "fruit: banana"]
```

### 2. **Stateful Batching Gatherer**

```java
public class BatchingGatherer<T> implements Gatherer<T, List<T>, List<T>> {
    private final int batchSize;
    
    public BatchingGatherer(int batchSize) {
        this.batchSize = batchSize;
    }
    
    @Override
    public Supplier<List<T>> initializer() {
        return ArrayList::new;
    }
    
    @Override
    public Integrator<List<T>, T, List<T>> integrator() {
        return (batch, element, downstream) -> {
            batch.add(element);
            if (batch.size() >= batchSize) {
                downstream.push(new ArrayList<>(batch));
                batch.clear();
            }
            return true; // Continue processing
        };
    }
    
    @Override
    public BiConsumer<List<T>, Downstream<? super List<T>>> finisher() {
        return (batch, downstream) -> {
            if (!batch.isEmpty()) {
                downstream.push(new ArrayList<>(batch));
            }
        };
    }
}

// Usage
List<List<Integer>> batches = Stream.of(1, 2, 3, 4, 5, 6, 7)
    .gather(new BatchingGatherer<>(3))
    .toList();
// Result: [[1, 2, 3], [4, 5, 6], [7]]
```

### 3. **Deduplication Gatherer**

```java
public class DeduplicationGatherer<T> implements Gatherer<T, Set<T>, T> {
    @Override
    public Supplier<Set<T>> initializer() {
        return HashSet::new;
    }
    
    @Override
    public Integrator<Set<T>, T, T> integrator() {
        return (seen, element, downstream) -> {
            if (seen.add(element)) {
                downstream.push(element);
            }
            return true;
        };
    }
    
    @Override
    public BinaryOperator<Set<T>> combiner() {
        return (set1, set2) -> {
            set1.addAll(set2);
            return set1;
        };
    }
}

// Usage
List<String> unique = Stream.of("a", "b", "a", "c", "b", "d")
    .gather(new DeduplicationGatherer<>())
    .toList();
// Result: ["a", "b", "c", "d"]
```

### 4. **Sliding Window with Custom Logic**

```java
public class SlidingWindowGatherer<T> implements Gatherer<T, List<T>, List<T>> {
    private final int windowSize;
    
    public SlidingWindowGatherer(int windowSize) {
        this.windowSize = windowSize;
    }
    
    @Override
    public Supplier<List<T>> initializer() {
        return ArrayList::new;
    }
    
    @Override
    public Integrator<List<T>, T, List<T>> integrator() {
        return (window, element, downstream) -> {
            window.add(element);
            if (window.size() > windowSize) {
                window.remove(0);
            }
            if (window.size() == windowSize) {
                downstream.push(new ArrayList<>(window));
            }
            return true;
        };
    }
    
    @Override
    public BinaryOperator<List<T>> combiner() {
        return (list1, list2) -> {
            // For parallel processing, merge the lists
            List<T> merged = new ArrayList<>(list1);
            merged.addAll(list2);
            return merged;
        };
    }
}
```

## Real-World Use Cases

### 1. **Time Series Analysis**

```java
public class TimeSeriesAnalyzer {
    public List<Double> calculateMovingAverage(List<Double> prices, int windowSize) {
        return prices.stream()
            .gather(new SlidingWindowGatherer<>(windowSize))
            .map(window -> window.stream().mapToDouble(Double::doubleValue).average().orElse(0.0))
            .toList();
    }
    
    public List<Double> detectAnomalies(List<Double> values, double threshold) {
        return values.stream()
            .gather(new SlidingWindowGatherer<>(5))
            .filter(window -> {
                double avg = window.stream().mapToDouble(Double::doubleValue).average().orElse(0.0);
                double current = window.get(window.size() - 1);
                return Math.abs(current - avg) > threshold;
            })
            .map(window -> window.get(window.size() - 1))
            .toList();
    }
}
```

### 2. **Log Processing**

```java
public class LogProcessor {
    public List<String> groupLogsBySession(List<LogEntry> logs) {
        return logs.stream()
            .gather(new BatchingGatherer<>(100)) // Process in batches of 100
            .map(batch -> processLogBatch(batch))
            .toList();
    }
    
    public List<String> findErrorPatterns(List<LogEntry> logs) {
        return logs.stream()
            .filter(log -> log.getLevel().equals("ERROR"))
            .gather(new SlidingWindowGatherer<>(3))
            .filter(window -> window.stream().allMatch(log -> log.getMessage().contains("timeout")))
            .map(window -> "Error pattern detected: " + window.size() + " consecutive timeouts")
            .toList();
    }
}
```

### 3. **Data Validation**

```java
public class DataValidator {
    public List<String> validateDataStream(List<DataRecord> records) {
        return records.stream()
            .gather(new ValidationGatherer())
            .toList();
    }
}

class ValidationGatherer implements Gatherer<DataRecord, ValidationState, String> {
    @Override
    public Supplier<ValidationState> initializer() {
        return ValidationState::new;
    }
    
    @Override
    public Integrator<ValidationState, DataRecord, String> integrator() {
        return (state, record, downstream) -> {
            if (state.isValid(record)) {
                state.addRecord(record);
                downstream.push("Valid: " + record.getId());
            } else {
                downstream.push("Invalid: " + record.getId() + " - " + state.getLastError());
            }
            return true;
        };
    }
}
```

## Advanced Patterns

### 1. **Chaining Gatherers**

```java
List<String> result = Stream.of("a", "b", "c", "d", "e", "f")
    .gather(new BatchingGatherer<>(2))           // Create batches of 2
    .gather(Gatherers.mapConcurrent(3, batch ->  // Process batches concurrently
        batch.stream().map(String::toUpperCase).toList()))
    .flatMap(List::stream)                       // Flatten back to individual elements
    .toList();
```

### 2. **Conditional Processing**

```java
public class ConditionalGatherer<T> implements Gatherer<T, Boolean, T> {
    private final Predicate<T> condition;
    
    public ConditionalGatherer(Predicate<T> condition) {
        this.condition = condition;
    }
    
    @Override
    public Supplier<Boolean> initializer() {
        return () -> false; // Start with condition not met
    }
    
    @Override
    public Integrator<Boolean, T, T> integrator() {
        return (conditionMet, element, downstream) -> {
            if (condition.test(element)) {
                conditionMet = true;
            }
            if (conditionMet) {
                downstream.push(element);
            }
            return true;
        };
    }
}
```

### 3. **Rate Limiting**

```java
public class RateLimitingGatherer<T> implements Gatherer<T, RateLimiter, T> {
    private final int maxPerSecond;
    
    public RateLimitingGatherer(int maxPerSecond) {
        this.maxPerSecond = maxPerSecond;
    }
    
    @Override
    public Supplier<RateLimiter> initializer() {
        return () -> RateLimiter.create(maxPerSecond);
    }
    
    @Override
    public Integrator<RateLimiter, T, T> integrator() {
        return (limiter, element, downstream) -> {
            limiter.acquire(); // Wait for permission
            downstream.push(element);
            return true;
        };
    }
}
```

## Performance Considerations

### 1. **Parallel Processing**

```java
// Enable parallel processing with combiner
public class ParallelGatherer<T> implements Gatherer<T, List<T>, T> {
    @Override
    public BinaryOperator<List<T>> combiner() {
        return (list1, list2) -> {
            list1.addAll(list2);
            return list1;
        };
    }
}

// Usage
List<String> result = Stream.of("a", "b", "c", "d", "e")
    .parallel()
    .gather(new ParallelGatherer<>())
    .toList();
```

### 2. **Memory Management**

```java
// Efficient memory usage for large datasets
public class MemoryEfficientGatherer<T> implements Gatherer<T, CircularBuffer<T>, T> {
    private final int maxSize;
    
    @Override
    public Supplier<CircularBuffer<T>> initializer() {
        return () -> new CircularBuffer<>(maxSize);
    }
    
    @Override
    public Integrator<CircularBuffer<T>, T, T> integrator() {
        return (buffer, element, downstream) -> {
            buffer.add(element);
            if (buffer.isFull()) {
                downstream.push(buffer.getOldest());
            }
            return true;
        };
    }
}
```

## Best Practices

### 1. **Keep Gatherers Stateless When Possible**

```java
// Good - stateless
static Gatherer<String, ?, String> toUpperCase() {
    return Gatherer.of((_, element, downstream) -> 
        downstream.push(element.toUpperCase()));
}

// Avoid - unnecessary state
static Gatherer<String, Integer, String> toUpperCaseWithCounter() {
    return Gatherer.of(
        () -> 0, // Unnecessary counter
        (counter, element, downstream) -> {
            downstream.push(element.toUpperCase());
            return counter + 1; // Unused state
        }
    );
}
```

### 2. **Handle Edge Cases**

```java
public class RobustGatherer<T> implements Gatherer<T, List<T>, List<T>> {
    @Override
    public Integrator<List<T>, T, List<T>> integrator() {
        return (state, element, downstream) -> {
            if (element != null) { // Handle null elements
                state.add(element);
            }
            return true;
        };
    }
    
    @Override
    public BiConsumer<List<T>, Downstream<? super List<T>>> finisher() {
        return (state, downstream) -> {
            if (!state.isEmpty()) { // Handle empty state
                downstream.push(new ArrayList<>(state));
            }
        };
    }
}
```

### 3. **Provide Meaningful Names**

```java
// Good - descriptive names
public class SlidingWindowGatherer<T> { ... }
public class BatchingGatherer<T> { ... }

// Avoid - generic names
public class MyGatherer<T> { ... }
public class CustomGatherer<T> { ... }
```

## Common Pitfalls

### 1. **Forgetting the Combiner for Parallel Streams**

```java
// Bad - won't work with parallel streams
public class BadGatherer<T> implements Gatherer<T, List<T>, T> {
    // Missing combiner() method
}

// Good - includes combiner
public class GoodGatherer<T> implements Gatherer<T, List<T>, T> {
    @Override
    public BinaryOperator<List<T>> combiner() {
        return (list1, list2) -> {
            list1.addAll(list2);
            return list1;
        };
    }
}
```

### 2. **Not Handling State Cleanup**

```java
// Bad - potential memory leak
public class LeakyGatherer<T> implements Gatherer<T, List<T>, T> {
    @Override
    public BiConsumer<List<T>, Downstream<? super T>> finisher() {
        return (state, downstream) -> {
            // Forgot to clear state
        };
    }
}

// Good - proper cleanup
public class CleanGatherer<T> implements Gatherer<T, List<T>, T> {
    @Override
    public BiConsumer<List<T>, Downstream<? super T>> finisher() {
        return (state, downstream) -> {
            state.clear(); // Clean up state
        };
    }
}
```

## Summary

Java's Stream Gatherers API provides a powerful way to create custom intermediate operations for streams. Key takeaways:

1. **Fills the gap** between simple operations and terminal collectors
2. **Enables stateful processing** within stream pipelines
3. **Supports parallel processing** with proper combiner implementation
4. **Composable and reusable** across different stream operations
5. **Perfect for complex data processing** patterns like windowing, batching, and stateful transformations

### When to Use Gatherers

- **Complex stateful transformations** that can't be done with standard operations
- **Windowing and batching** operations
- **Custom grouping** and aggregation patterns
- **Data validation** and processing pipelines
- **Performance-critical** stream processing

### When to Stick with Standard Operations

- **Simple transformations** that can be done with `map`, `filter`, etc.
- **Basic aggregation** that can be done with `collect()`
- **One-time operations** that don't need to be reusable

Remember: Gatherers are a powerful tool, but use them judiciously. They're perfect for complex scenarios where the standard Stream API falls short, but don't over-engineer simple problems!

*This post covers the Stream Gatherers API as introduced in Java 24 and finalized in Java 25. The API provides a flexible way to create custom intermediate operations for Java streams.*
