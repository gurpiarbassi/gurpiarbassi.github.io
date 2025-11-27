---
layout: post
title: "JUnit @RepeatedTest: Running Tests Multiple Times"
published: true
date: 2025-11-27
categories: java testing junit
---

# Motivation

In this post, we'll explore JUnit's **@RepeatedTest** annotation - a powerful feature that allows you to run the same test multiple times. This is especially useful for testing non-deterministic behavior, performance testing, stress testing, and ensuring reliability in concurrent or timing-sensitive scenarios.

## The Problem: Testing Non-Deterministic Behavior

### Traditional Approach

```java
// Old way - manually repeating test logic
@Test
public void testRandomNumberGeneration() {
    for (int i = 0; i < 10; i++) {
        int random = generateRandomNumber();
        assertTrue(random >= 0 && random < 100);
    }
}

// Old way - multiple test methods
@Test
public void testConnection1() { testConnection(); }
@Test
public void testConnection2() { testConnection(); }
@Test
public void testConnection3() { testConnection(); }

private void testConnection() {
    // Test logic here
}
```

### The Gap

- **Manual loops**: Clutters test code with iteration logic
- **Multiple test methods**: Duplicates code and makes test reports confusing
- **No test isolation**: Each iteration shares state
- **Poor reporting**: Can't see individual iteration results
- **Missing**: A clean way to repeat tests with proper isolation and reporting

## What is @RepeatedTest?

**@RepeatedTest** is a JUnit 5 annotation that allows you to run a test method multiple times, with each execution being treated as a separate test. This provides proper test isolation, better reporting, and cleaner test code.

### Key Characteristics

- **Multiple executions**: Run the same test N times
- **Test isolation**: Each repetition is independent
- **Better reporting**: See results for each iteration
- **Custom naming**: Customize test display names
- **Repetition info**: Access repetition metadata in tests

## @RepeatedTest API

### Basic Usage

```java
import org.junit.jupiter.api.RepeatedTest;
import org.junit.jupiter.api.RepetitionInfo;

class RepeatedTestExample {
    
    @RepeatedTest(5)
    void testRandomNumberGeneration() {
        int random = generateRandomNumber();
        assertTrue(random >= 0 && random < 100);
    }
    
    @RepeatedTest(value = 10, name = "Running test {currentRepetition}/{totalRepetitions}")
    void testWithCustomName() {
        // Test logic
    }
}
```

### Accessing Repetition Information

```java
@RepeatedTest(5)
void testWithRepetitionInfo(RepetitionInfo repetitionInfo) {
    int currentRepetition = repetitionInfo.getCurrentRepetition();
    int totalRepetitions = repetitionInfo.getTotalRepetions();
    
    System.out.println("Repetition " + currentRepetition + " of " + totalRepetitions);
    
    // Test logic that might depend on repetition number
    if (currentRepetition == 1) {
        // First iteration setup
    }
}
```

## Real-World Use Cases

### 1. **Testing Non-Deterministic Behavior**

```java
class RandomNumberTest {
    
    @RepeatedTest(100)
    void testRandomNumberInRange() {
        int random = ThreadLocalRandom.current().nextInt(0, 100);
        assertTrue(random >= 0 && random < 100, 
            "Random number should be in range [0, 100)");
    }
    
    @RepeatedTest(50)
    void testUUIDGeneration() {
        String uuid = UUID.randomUUID().toString();
        assertNotNull(uuid);
        assertEquals(36, uuid.length());
        assertTrue(uuid.matches("[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}"));
    }
}
```

### 2. **Concurrency Testing**

```java
class ConcurrencyTest {
    
    @RepeatedTest(20)
    void testThreadSafety(RepetitionInfo info) {
        Counter counter = new Counter();
        ExecutorService executor = Executors.newFixedThreadPool(10);
        
        List<Future<?>> futures = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            futures.add(executor.submit(() -> counter.increment()));
        }
        
        futures.forEach(future -> {
            try {
                future.get();
            } catch (Exception e) {
                fail("Thread execution failed: " + e.getMessage());
            }
        });
        
        executor.shutdown();
        assertEquals(100, counter.getValue(), 
            "Counter should be 100 after 100 increments (Repetition " + 
            info.getCurrentRepetition() + ")");
    }
}
```

### 3. **Performance and Stress Testing**

```java
class PerformanceTest {
    
    @RepeatedTest(10)
    void testResponseTime(RepetitionInfo info) {
        long startTime = System.currentTimeMillis();
        
        // Simulate API call
        String response = apiClient.get("/api/users");
        
        long duration = System.currentTimeMillis() - startTime;
        
        assertNotNull(response);
        assertTrue(duration < 1000, 
            "Response time should be under 1 second. Actual: " + duration + "ms " +
            "(Repetition " + info.getCurrentRepetition() + ")");
    }
    
    @RepeatedTest(value = 5, name = "Load test iteration {currentRepetition}")
    void testUnderLoad() {
        // Simulate load testing
        List<CompletableFuture<String>> futures = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            futures.add(CompletableFuture.supplyAsync(() -> 
                apiClient.get("/api/data")));
        }
        
        futures.forEach(future -> {
            assertDoesNotThrow(() -> future.get(5, TimeUnit.SECONDS));
        });
    }
}
```

### 4. **Flaky Test Detection**

```java
class FlakyTestDetection {
    
    @RepeatedTest(50)
    void testDatabaseConnection(RepetitionInfo info) {
        // This test might fail occasionally due to network issues
        try {
            Connection connection = database.getConnection();
            assertTrue(connection.isValid(5));
            connection.close();
        } catch (SQLException e) {
            fail("Database connection failed on repetition " + 
                 info.getCurrentRepetition() + ": " + e.getMessage());
        }
    }
    
    @RepeatedTest(100)
    void testRaceCondition(RepetitionInfo info) {
        // Test for race conditions that might not appear every time
        SharedResource resource = new SharedResource();
        ExecutorService executor = Executors.newFixedThreadPool(5);
        
        List<Future<?>> futures = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            futures.add(executor.submit(() -> resource.modify()));
        }
        
        futures.forEach(future -> assertDoesNotThrow(() -> future.get()));
        executor.shutdown();
        
        assertTrue(resource.isConsistent(), 
            "Resource should be consistent after all operations " +
            "(Repetition " + info.getCurrentRepetition() + ")");
    }
}
```

### 5. **Testing with Different Data Sets**

```java
class DataVariationTest {
    
    @RepeatedTest(10)
    void testWithDifferentInputs(RepetitionInfo info) {
        // Generate different test data for each repetition
        int input = info.getCurrentRepetition() * 10;
        int result = calculator.process(input);
        
        assertTrue(result > 0, 
            "Result should be positive for input " + input);
    }
    
    @RepeatedTest(20)
    void testSortingAlgorithm(RepetitionInfo info) {
        // Test with different array sizes and contents
        int size = info.getCurrentRepetition() * 5;
        int[] array = generateRandomArray(size);
        
        int[] sorted = sortingAlgorithm.sort(array);
        
        assertArrayEquals(Arrays.stream(array).sorted().toArray(), sorted,
            "Array should be sorted correctly (size: " + size + ")");
    }
}
```

## Advanced Usage Patterns

### 1. **Conditional Repetition Based on Environment**

```java
class ConditionalRepeatedTest {
    
    @RepeatedTest(value = 100, name = "Stress test {currentRepetition}/{totalRepetitions}")
    void testUnderStress(RepetitionInfo info) {
        // Only run many repetitions in CI environment
        if (System.getenv("CI") == null && info.getCurrentRepetition() > 10) {
            return; // Skip remaining repetitions in local environment
        }
        
        // Test logic
        performStressTest();
    }
}
```

### 2. **Combining with Other Annotations**

```java
class CombinedAnnotationsTest {
    
    @RepeatedTest(5)
    @DisplayName("API endpoint reliability test")
    @Tag("integration")
    void testApiReliability(RepetitionInfo info) {
        // Test logic
    }
    
    @RepeatedTest(10)
    @Timeout(value = 5, unit = TimeUnit.SECONDS)
    void testWithTimeout(RepetitionInfo info) {
        // Each repetition must complete within 5 seconds
        performTimeSensitiveOperation();
    }
    
    @RepeatedTest(3)
    @Disabled("Temporarily disabled for debugging")
    void testCurrentlyDisabled() {
        // This test won't run
    }
}
```

### 3. **Dynamic Repetition Count**

```java
class DynamicRepetitionTest {
    
    @RepeatedTest(value = 1) // Will be overridden
    void testWithDynamicRepetitions(RepetitionInfo info) {
        int repetitions = Integer.parseInt(
            System.getProperty("test.repetitions", "10"));
        
        if (info.getCurrentRepetition() < repetitions) {
            // Continue testing
            performTest();
        }
    }
    
    // Better approach: Use @ParameterizedTest with @ValueSource
    @ParameterizedTest
    @ValueSource(ints = {1, 5, 10, 20, 50})
    void testWithParameterizedRepetitions(int repetition) {
        for (int i = 0; i < repetition; i++) {
            performTest();
        }
    }
}
```

### 4. **Custom Test Naming**

```java
class CustomNamingTest {
    
    @RepeatedTest(
        value = 5,
        name = "Attempt {currentRepetition} of {totalRepetitions} - Testing retry logic"
    )
    void testRetryMechanism(RepetitionInfo info) {
        // Test retry logic
    }
    
    @RepeatedTest(
        value = 10,
        name = "Performance run #{currentRepetition} (Total: {totalRepetitions})"
    )
    void testPerformance(RepetitionInfo info) {
        // Performance test
    }
    
    @RepeatedTest(
        value = 100,
        name = DisplayName.PLACEHOLDER + " - " + 
               "Repetition {currentRepetition}/{totalRepetitions}"
    )
    @DisplayName("Concurrency safety test")
    void testConcurrency() {
        // Concurrency test
    }
}
```

## Best Practices

### 1. **Use Appropriate Repetition Counts**

```java
// Good - reasonable count for non-deterministic tests
@RepeatedTest(50)
void testRandomBehavior() { }

// Good - higher count for critical concurrency tests
@RepeatedTest(100)
void testThreadSafety() { }

// Avoid - too many repetitions slow down test suite
@RepeatedTest(10000)
void testSomething() { }

// Better - use parameterized tests for systematic testing
@ParameterizedTest
@ValueSource(ints = {10, 50, 100, 500})
void testWithDifferentCounts(int count) { }
```

### 2. **Provide Meaningful Test Names**

```java
// Good - descriptive name
@RepeatedTest(value = 20, name = "Testing API reliability - Run {currentRepetition}/{totalRepetitions}")
void testApiReliability() { }

// Avoid - generic name
@RepeatedTest(20)
void test() { }
```

### 3. **Use RepetitionInfo for Context**

```java
// Good - use repetition info for better error messages
@RepeatedTest(10)
void testWithContext(RepetitionInfo info) {
    try {
        performOperation();
    } catch (Exception e) {
        fail("Failed on repetition " + info.getCurrentRepetition() + 
             " of " + info.getTotalRepetions() + ": " + e.getMessage());
    }
}
```

### 4. **Combine with Assertions for Better Reporting**

```java
// Good - clear assertion messages
@RepeatedTest(50)
void testWithClearMessages(RepetitionInfo info) {
    int result = performOperation();
    assertTrue(result > 0, 
        "Result should be positive (Repetition " + 
        info.getCurrentRepetition() + ")");
}
```

## Common Pitfalls

### 1. **Shared State Between Repetitions**

```java
// Bad - shared state can cause issues
class BadRepeatedTest {
    private int counter = 0; // Shared across all repetitions!
    
    @RepeatedTest(10)
    void testWithSharedState() {
        counter++;
        assertEquals(1, counter); // Will fail after first repetition
    }
}

// Good - each repetition is isolated
class GoodRepeatedTest {
    @RepeatedTest(10)
    void testIsolated() {
        int counter = 0; // Fresh for each repetition
        counter++;
        assertEquals(1, counter); // Always passes
    }
}
```

### 2. **Assuming Execution Order**

```java
// Bad - don't assume repetitions run in order
@RepeatedTest(10)
void testWithOrderAssumption(RepetitionInfo info) {
    if (info.getCurrentRepetition() == 1) {
        setup(); // Might not be first!
    }
}

// Good - use @BeforeEach for setup
@BeforeEach
void setUp() {
    // Runs before each repetition
}

@RepeatedTest(10)
void testWithProperSetup() {
    // Test logic
}
```

### 3. **Too Many Repetitions**

```java
// Bad - slows down test suite unnecessarily
@RepeatedTest(10000)
void testSomething() {
    // Simple test that doesn't need 10000 repetitions
}

// Good - appropriate repetition count
@RepeatedTest(10)
void testSomething() {
    // Test logic
}
```

### 4. **Not Using RepetitionInfo for Debugging**

```java
// Bad - hard to debug which repetition failed
@RepeatedTest(100)
void testWithoutContext() {
    performOperation();
}

// Good - includes context in error messages
@RepeatedTest(100)
void testWithContext(RepetitionInfo info) {
    try {
        performOperation();
    } catch (Exception e) {
        fail("Failed on repetition " + info.getCurrentRepetition() + 
             ": " + e.getMessage());
    }
}
```

## Comparison: @RepeatedTest vs Alternatives

| Approach | Use Case | Pros | Cons |
|----------|----------|------|------|
| **@RepeatedTest** | Non-deterministic, concurrency, flaky tests | Clean syntax, proper isolation, good reporting | Fixed repetition count |
| **Manual loop** | Simple repeated logic | Simple | No isolation, poor reporting |
| **@ParameterizedTest** | Different inputs | Flexible, systematic | More verbose |
| **Multiple @Test methods** | Different test scenarios | Clear separation | Code duplication |

## Summary

JUnit's `@RepeatedTest` annotation provides a clean and powerful way to run tests multiple times. Key takeaways:

1. **Perfect for non-deterministic behavior** - Random, timing, or concurrency-related tests
2. **Proper test isolation** - Each repetition is independent
3. **Better reporting** - See results for each iteration
4. **Access repetition metadata** - Use `RepetitionInfo` for context
5. **Customizable naming** - Make test reports more readable

### When to Use @RepeatedTest

- **Non-deterministic tests** - Random number generation, UUID creation
- **Concurrency testing** - Thread safety, race conditions
- **Flaky test detection** - Finding intermittent failures
- **Performance testing** - Response time validation
- **Stress testing** - System behavior under load

### When to Use Alternatives

- **Different inputs** - Use `@ParameterizedTest`
- **Different test scenarios** - Use multiple `@Test` methods
- **Simple repeated logic** - Use a loop within a single test
- **Systematic testing** - Use `@ParameterizedTest` with `@ValueSource`

Remember: `@RepeatedTest` is a powerful tool for ensuring reliability, but use it judiciously. Too many repetitions can slow down your test suite, and it's not a substitute for fixing flaky tests - it helps you find them!

