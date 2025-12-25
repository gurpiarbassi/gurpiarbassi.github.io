---
layout: post
title: "Java 25 Scoped Values: Solving ThreadLocal Problems"
published: true
date: 2025-11-29
categories: java programming concurrency
---

# Motivation

In this post, we'll explore Java 25's **Scoped Values** - a modern replacement for `ThreadLocal` that solves critical problems with thread-local storage, especially in thread pool environments. Scoped Values provide a safer, more efficient way to pass context data through call chains without the pitfalls of traditional `ThreadLocal` usage.

## The Problem: ThreadLocal in Thread Pools

### Traditional ThreadLocal Usage

```java
public class UserContext {
    private static final ThreadLocal<String> userId = new ThreadLocal<>();
    private static final ThreadLocal<String> requestId = new ThreadLocal<>();
    
    public static void setUserId(String id) {
        userId.set(id);
    }
    
    public static String getUserId() {
        return userId.get();
    }
    
    public static void setRequestId(String id) {
        requestId.set(id);
    }
    
    public static String getRequestId() {
        return requestId.get();
    }
    
    // CRITICAL: Must call this after use!
    public static void clear() {
        userId.remove();
        requestId.remove();
    }
}
```

### The Critical Issues

#### 1. **Thread Pool Reuse and Data Leakage**

The most serious problem with `ThreadLocal` in thread pools is that threads are reused across multiple requests. If you forget to call `remove()`, data from one request can leak into another:

```java
// Request 1 - Thread A
UserContext.setUserId("user-123");
processRequest(); // Inside processRequest(), it can call UserContext.getUserId()
                  // because getUserId() is static (can be called from anywhere)
                  // and the ThreadLocal value is stored on this thread
// FORGOT TO CALL UserContext.clear()!
// The ThreadLocal still contains "user-123" on Thread A

// Example of what processRequest() might look like:
void processRequest() {
    String userId = UserContext.getUserId(); // Returns "user-123"
    // Process request with userId...
}

// Request 2 - Thread A (reused from pool)
UserContext.setUserId("user-456"); // Overwrites "user-123" on Thread A
String userId = UserContext.getUserId(); // Returns "user-456" - OK

// Request 3 - Thread A (reused again)
// No setUserId() called, but...
String userId = UserContext.getUserId(); // Returns "user-456" - WRONG!
// This is data from Request 2 leaking into Request 3!
// The ThreadLocal still contains "user-456" from the previous request
```

#### 2. **Forgetting to Call remove()**

```java
public void handleRequest(HttpRequest request) {
    try {
        UserContext.setUserId(request.getUserId());
        UserContext.setRequestId(request.getRequestId());
        
        processRequest();
        
        // Easy to forget this!
        // UserContext.clear(); // MISSING!
    } catch (Exception e) {
        // Even if we remember in the happy path,
        // exceptions can bypass cleanup
        // UserContext.clear(); // MISSING!
        throw e;
    }
}
```

#### 3. **Exception Handling Complexity**

```java
public void handleRequest(HttpRequest request) {
    UserContext.setUserId(request.getUserId());
    try {
        processRequest();
    } finally {
        // Must remember finally block
        UserContext.clear();
    }
}
```

#### 4. **Nested Context Issues**

```java
public void outerMethod() {
    UserContext.setUserId("outer-user");
    try {
        innerMethod();
    } finally {
        UserContext.clear(); // Clears "outer-user" even if innerMethod set something
    }
}

public void innerMethod() {
    UserContext.setUserId("inner-user");
    // If innerMethod throws, outerMethod's finally clears everything
    // But what if we wanted to preserve outer context?
}
```

#### 5. **Memory Leaks in Long-Lived Threads**

```java
// Thread pool with long-lived threads
ExecutorService executor = Executors.newFixedThreadPool(10);

for (int i = 0; i < 1000; i++) {
    executor.submit(() -> {
        UserContext.setUserId("user-" + ThreadLocalRandom.current().nextInt());
        processRequest();
        // If remove() is forgotten, ThreadLocal map grows indefinitely
        // Each thread maintains its own ThreadLocal map
    });
}
```

## What are Scoped Values?

**Scoped Values** provide a structured way to pass immutable data through a call chain. Unlike `ThreadLocal`, they are automatically cleaned up when the scope exits, preventing data leakage and memory issues.

### Key Characteristics

- **Automatic cleanup**: Values are automatically removed when scope exits
- **Immutable**: Once set, values cannot be changed
- **Structured**: Values are bound to a specific scope (method call, lambda, etc.)
- **Thread-safe**: Safe for concurrent access
- **No manual cleanup**: No need to call `remove()`

## Scoped Values API

### Basic Usage

```java
import java.util.concurrent.StructuredTaskScope;
import java.lang.ScopedValue;

public class UserContext {
    // Define a scoped value
    private static final ScopedValue<String> USER_ID = ScopedValue.newInstance();
    private static final ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();
    
    // Run code with scoped values
    public static void runWithContext(String userId, String requestId, Runnable task) {
        ScopedValue.where(USER_ID, userId)
                   .where(REQUEST_ID, requestId)
                   .run(task);
    }
    
    // Access scoped values
    public static String getUserId() {
        return USER_ID.get();
    }
    
    public static String getRequestId() {
        return REQUEST_ID.get();
    }
}
```

### Example: Request Handling

```java
public class RequestHandler {
    public void handleRequest(HttpRequest request) {
        UserContext.runWithContext(
            request.getUserId(),
            request.getRequestId(),
            () -> {
                // All code here has access to USER_ID and REQUEST_ID
                processRequest();
                logRequest();
                updateMetrics();
            }
        );
        // Values are automatically cleared here - no manual cleanup needed!
    }
    
    private void processRequest() {
        // Works because processRequest() is called from within the scope
        // created by ScopedValue.where(...).run(...) above
        // The scoped value is bound to the current thread's execution context
        String userId = UserContext.getUserId(); // Works!
        String requestId = UserContext.getRequestId(); // Works!
        // Process request...
    }
    
    private void logRequest() {
        // Also works - called from within the same scope
        String userId = UserContext.getUserId(); // Still works!
        logger.info("Processing request for user: " + userId);
    }
}
```

**Important Note**: `processRequest()` and `logRequest()` can access the scoped values not because `getUserId()` is static (though that makes it callable from anywhere), but because they are called from **within the scope** created by `ScopedValue.where(...).run(...)`. The scoped value is bound to the current thread's execution context, so any method called within that scope (directly or indirectly) can access the bound values. If you tried to call `UserContext.getUserId()` outside the scope (after the `run()` completes), it would throw an exception.

### Nested Scopes

```java
public class NestedScopesExample {
    private static final ScopedValue<String> OUTER_VALUE = ScopedValue.newInstance();
    private static final ScopedValue<String> INNER_VALUE = ScopedValue.newInstance();
    
    public void outerMethod() {
        ScopedValue.where(OUTER_VALUE, "outer")
                   .run(() -> {
                       System.out.println(OUTER_VALUE.get()); // "outer"
                       
                       innerMethod();
                       
                       System.out.println(OUTER_VALUE.get()); // Still "outer"
                   });
    }
    
    public void innerMethod() {
        ScopedValue.where(INNER_VALUE, "inner")
                   .run(() -> {
                       System.out.println(OUTER_VALUE.get()); // "outer" - still accessible!
                       System.out.println(INNER_VALUE.get()); // "inner"
                   });
        // INNER_VALUE is cleared here, but OUTER_VALUE remains
    }
}
```

## Real-World Use Cases

### 1. **Web Request Context**

```java
public class WebRequestContext {
    private static final ScopedValue<String> USER_ID = ScopedValue.newInstance();
    private static final ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();
    private static final ScopedValue<Locale> LOCALE = ScopedValue.newInstance();
    private static final ScopedValue<HttpHeaders> HEADERS = ScopedValue.newInstance();
    
    public static void runWithRequest(HttpRequest request, Runnable handler) {
        ScopedValue.where(USER_ID, request.getUserId())
                   .where(REQUEST_ID, request.getRequestId())
                   .where(LOCALE, request.getLocale())
                   .where(HEADERS, request.getHeaders())
                   .run(handler);
    }
    
    public static String getUserId() {
        return USER_ID.get();
    }
    
    public static String getRequestId() {
        return REQUEST_ID.get();
    }
    
    public static Locale getLocale() {
        return LOCALE.get();
    }
}

// Usage in servlet/framework
public void doGet(HttpServletRequest request, HttpServletResponse response) {
    WebRequestContext.runWithRequest(
        new HttpRequest(request),
        () -> {
            // All handlers have access to context
            controller.handle();
        }
    );
}
```

### 2. **Database Transaction Context**

```java
public class TransactionContext {
    private static final ScopedValue<Connection> CONNECTION = ScopedValue.newInstance();
    private static final ScopedValue<Transaction> TRANSACTION = ScopedValue.newInstance();
    
    public static <T> T runInTransaction(Function<Connection, T> operation) {
        return ScopedValue.where(CONNECTION, getConnection())
                          .where(TRANSACTION, beginTransaction())
                          .call(() -> {
                              try {
                                  T result = operation.apply(CONNECTION.get());
                                  TRANSACTION.get().commit();
                                  return result;
                              } catch (Exception e) {
                                  TRANSACTION.get().rollback();
                                  throw e;
                              } finally {
                                  CONNECTION.get().close();
                              }
                          });
    }
    
    public static Connection getConnection() {
        return CONNECTION.get();
    }
}

// Usage
public User createUser(String name, String email) {
    return TransactionContext.runInTransaction(connection -> {
        // Connection is automatically available
        return userRepository.save(connection, new User(name, email));
    });
}
```

### 3. **Security Context**

```java
public class SecurityContext {
    private static final ScopedValue<Principal> PRINCIPAL = ScopedValue.newInstance();
    private static final ScopedValue<Set<String>> ROLES = ScopedValue.newInstance();
    private static final ScopedValue<Set<String>> PERMISSIONS = ScopedValue.newInstance();
    
    public static void runAs(Principal principal, Runnable task) {
        Set<String> roles = loadRoles(principal);
        Set<String> permissions = loadPermissions(principal);
        
        ScopedValue.where(PRINCIPAL, principal)
                   .where(ROLES, roles)
                   .where(PERMISSIONS, permissions)
                   .run(task);
    }
    
    public static Principal getPrincipal() {
        return PRINCIPAL.get();
    }
    
    public static boolean hasRole(String role) {
        return ROLES.get().contains(role);
    }
    
    public static boolean hasPermission(String permission) {
        return PERMISSIONS.get().contains(permission);
    }
}

// Usage
public void performSecureOperation() {
    SecurityContext.runAs(currentUser, () -> {
        if (SecurityContext.hasRole("ADMIN")) {
            performAdminOperation();
        }
    });
}
```

### 4. **Logging Context**

```java
public class LoggingContext {
    private static final ScopedValue<String> CORRELATION_ID = ScopedValue.newInstance();
    private static final ScopedValue<String> TRACE_ID = ScopedValue.newInstance();
    private static final ScopedValue<Map<String, String>> MDC = ScopedValue.newInstance();
    
    public static void runWithLoggingContext(String correlationId, String traceId, Runnable task) {
        Map<String, String> mdc = new HashMap<>();
        mdc.put("correlationId", correlationId);
        mdc.put("traceId", traceId);
        
        ScopedValue.where(CORRELATION_ID, correlationId)
                   .where(TRACE_ID, traceId)
                   .where(MDC, Map.copyOf(mdc))
                   .run(task);
    }
    
    public static String getCorrelationId() {
        return CORRELATION_ID.get();
    }
    
    public static void log(String message) {
        String correlationId = getCorrelationId();
        logger.info("[{}] {}", correlationId, message);
    }
}
```

### 5. **Testing Context**

```java
public class TestContext {
    private static final ScopedValue<TestEnvironment> ENV = ScopedValue.newInstance();
    private static final ScopedValue<MockDatabase> MOCK_DB = ScopedValue.newInstance();
    
    public static void runInTestEnvironment(Runnable test) {
        TestEnvironment env = new TestEnvironment();
        MockDatabase mockDb = new MockDatabase();
        
        ScopedValue.where(ENV, env)
                   .where(MOCK_DB, mockDb)
                   .run(() -> {
                       try {
                           test.run();
                       } finally {
                           env.cleanup();
                           mockDb.cleanup();
                       }
                   });
    }
    
    public static TestEnvironment getEnvironment() {
        return ENV.get();
    }
    
    public static MockDatabase getMockDatabase() {
        return MOCK_DB.get();
    }
}
```

## Comparison: ThreadLocal vs Scoped Values

| Feature | ThreadLocal | Scoped Values |
|---------|-------------|---------------|
| **Cleanup** | Manual (`remove()`) | Automatic |
| **Data Leakage** | High risk in thread pools | Protected by scope |
| **Exception Safety** | Requires try-finally | Automatic |
| **Nested Contexts** | Overwrites or requires careful management | Proper nesting support |
| **Memory Leaks** | Possible if `remove()` forgotten | Automatic cleanup prevents leaks |
| **Immutability** | Mutable | Immutable |
| **Structured Concurrency** | Not compatible | Compatible |
| **Performance** | Good | Better (optimized for structured concurrency) |

## Thread Pool Example: Before and After

### Before: ThreadLocal (Problematic)

```java
public class RequestProcessor {
    private static final ThreadLocal<String> USER_ID = new ThreadLocal<>();
    private ExecutorService executor = Executors.newFixedThreadPool(10);
    
    public void processRequest(HttpRequest request) {
        executor.submit(() -> {
            USER_ID.set(request.getUserId());
            try {
                process();
                log();
            } finally {
                // Easy to forget!
                USER_ID.remove();
            }
        });
    }
    
    private void process() {
        String userId = USER_ID.get(); // Could be null or wrong user!
        // Process...
    }
    
    private void log() {
        String userId = USER_ID.get(); // Could be null or wrong user!
        logger.info("User: " + userId);
    }
}
```

### After: Scoped Values (Safe)

```java
public class RequestProcessor {
    private static final ScopedValue<String> USER_ID = ScopedValue.newInstance();
    private ExecutorService executor = Executors.newFixedThreadPool(10);
    
    public void processRequest(HttpRequest request) {
        executor.submit(() -> {
            ScopedValue.where(USER_ID, request.getUserId())
                       .run(() -> {
                           process();
                           log();
                       });
            // Automatic cleanup - no manual remove() needed!
        });
    }
    
    private void process() {
        String userId = USER_ID.get(); // Always correct for this scope!
        // Process...
    }
    
    private void log() {
        String userId = USER_ID.get(); // Always correct for this scope!
        logger.info("User: " + userId);
    }
}
```

## Advanced Usage Patterns

### 1. **Combining with Structured Concurrency**

```java
public class StructuredConcurrencyExample {
    private static final ScopedValue<String> USER_ID = ScopedValue.newInstance();
    
    public void processWithStructuredConcurrency(String userId) {
        ScopedValue.where(USER_ID, userId)
                   .run(() -> {
                       try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
                           Future<String> userData = scope.fork(() -> fetchUserData());
                           Future<List<Order>> orders = scope.fork(() -> fetchOrders());
                           
                           scope.join();
                           
                           // Both tasks have access to USER_ID
                           String data = userData.resultNow();
                           List<Order> orderList = orders.resultNow();
                       }
                   });
    }
    
    private String fetchUserData() {
        String userId = USER_ID.get(); // Available in forked task!
        return userService.getUserData(userId);
    }
    
    private List<Order> fetchOrders() {
        String userId = USER_ID.get(); // Available in forked task!
        return orderService.getOrders(userId);
    }
}
```

### 2. **Conditional Access**

```java
public class ConditionalAccessExample {
    private static final ScopedValue<String> USER_ID = ScopedValue.newInstance();
    
    public void process() {
        ScopedValue.where(USER_ID, "user-123")
                   .run(() -> {
                       // Check if value is available
                       if (USER_ID.isBound()) {
                           String userId = USER_ID.get();
                           processWithUser(userId);
                       } else {
                           processWithoutUser();
                       }
                   });
    }
}
```

### 3. **Default Values**

```java
public class DefaultValueExample {
    private static final ScopedValue<String> USER_ID = ScopedValue.newInstance();
    
    public void process() {
        // Process with user ID if available
        String userId = USER_ID.orElse("anonymous");
        processRequest(userId);
    }
    
    public void processWithFallback() {
        String userId = USER_ID.orElseThrow(() -> 
            new IllegalStateException("User ID required"));
        processRequest(userId);
    }
}
```

## Best Practices

### 1. **Always Use Scoped Values in Thread Pools**

```java
// Good - automatic cleanup
executor.submit(() -> {
    ScopedValue.where(USER_ID, userId).run(() -> {
        processRequest();
    });
});

// Avoid - manual cleanup required
executor.submit(() -> {
    USER_ID.set(userId);
    try {
        processRequest();
    } finally {
        USER_ID.remove(); // Easy to forget!
    }
});
```

### 2. **Use Descriptive Names**

```java
// Good - clear purpose
private static final ScopedValue<String> REQUEST_CORRELATION_ID = ScopedValue.newInstance();

// Avoid - unclear purpose
private static final ScopedValue<String> ID = ScopedValue.newInstance();
```

### 3. **Keep Values Immutable**

```java
// Good - immutable value
ScopedValue.where(USER_ID, userId) // String is immutable
           .run(() -> { });

// Good - immutable collection
ScopedValue.where(ROLES, Set.copyOf(roles)) // Immutable set
           .run(() -> { });

// Avoid - mutable objects
ScopedValue.where(USER_DATA, userData) // If userData is mutable, changes affect scope
           .run(() -> { });
```

### 4. **Handle Missing Values Gracefully**

```java
// Good - provide fallback
String userId = USER_ID.orElse("anonymous");

// Good - fail fast if required
String userId = USER_ID.orElseThrow(() -> 
    new IllegalStateException("User ID required"));

// Avoid - null checks everywhere
String userId = USER_ID.get(); // Throws if not bound
if (userId != null) { // This check is unnecessary
    // ...
}
```

## Common Pitfalls

### 1. **Trying to Modify Scoped Values**

```java
// Bad - ScopedValue is immutable
ScopedValue.where(USER_ID, "user-123")
           .run(() -> {
               // This doesn't exist - values are immutable!
               // USER_ID.set("user-456"); // Compile error!
           });

// Good - create new scope if needed
ScopedValue.where(USER_ID, "user-123")
           .run(() -> {
               ScopedValue.where(USER_ID, "user-456")
                          .run(() -> {
                              // New scope with different value
                          });
           });
```

### 2. **Accessing Outside Scope**

```java
// Bad - accessing outside scope
ScopedValue.where(USER_ID, "user-123")
           .run(() -> {
               // ...
           });
String userId = USER_ID.get(); // Throws ScopedValue.CarrierException!

// Good - access within scope
ScopedValue.where(USER_ID, "user-123")
           .run(() -> {
               String userId = USER_ID.get(); // OK
           });
```

### 3. **Not Using in Thread Pools**

```java
// Bad - ThreadLocal in thread pool
executor.submit(() -> {
    USER_ID.set(userId);
    process();
    // Forgot remove() - data leakage!
});

// Good - ScopedValue in thread pool
executor.submit(() -> {
    ScopedValue.where(USER_ID, userId)
               .run(() -> {
                   process();
               });
});
```

### 4. **Storing Mutable Objects**

```java
// Bad - mutable object
Map<String, String> context = new HashMap<>();
ScopedValue.where(CONTEXT, context)
           .run(() -> {
               context.put("key", "value"); // Modifies shared state!
           });

// Good - immutable object
Map<String, String> context = Map.of("key", "value");
ScopedValue.where(CONTEXT, context)
           .run(() -> {
               // context is immutable
           });
```

## Migration Guide: ThreadLocal to Scoped Values

### Step 1: Replace ThreadLocal Declaration

```java
// Before
private static final ThreadLocal<String> USER_ID = new ThreadLocal<>();

// After
private static final ScopedValue<String> USER_ID = ScopedValue.newInstance();
```

### Step 2: Replace set() with where().run()

```java
// Before
USER_ID.set(userId);
try {
    process();
} finally {
    USER_ID.remove();
}

// After
ScopedValue.where(USER_ID, userId)
           .run(() -> {
               process();
           });
```

### Step 3: Replace get() Calls

```java
// Before
String userId = USER_ID.get(); // Can return null

// After
String userId = USER_ID.get(); // Throws if not bound
// Or with fallback:
String userId = USER_ID.orElse("default");
```

### Step 4: Update Exception Handling

```java
// Before
USER_ID.set(userId);
try {
    process();
} catch (Exception e) {
    handleError();
} finally {
    USER_ID.remove(); // Must remember!
}

// After
ScopedValue.where(USER_ID, userId)
           .run(() -> {
               try {
                   process();
               } catch (Exception e) {
                   handleError();
               }
           });
// Automatic cleanup - no finally needed!
```

## Summary

Java 25's Scoped Values provide a modern, safe alternative to `ThreadLocal` that solves critical problems:

1. **Automatic cleanup** - No need to call `remove()`
2. **Thread pool safety** - Prevents data leakage when threads are reused
3. **Exception safety** - Values are cleaned up even if exceptions occur
4. **Structured concurrency** - Works seamlessly with structured task scopes
5. **Immutable values** - Prevents accidental modifications
6. **Better performance** - Optimized for modern concurrency patterns

### When to Use Scoped Values

- **Thread pool environments** - Web servers, async frameworks
- **Request context** - User ID, request ID, correlation IDs
- **Security context** - Principal, roles, permissions
- **Transaction context** - Database connections, transactions
- **Logging context** - MDC, trace IDs
- **Testing** - Test environment setup

### When to Still Use ThreadLocal

- **Legacy code** - Existing codebases not yet migrated
- **JDK < 21** - Scoped Values require Java 21+ (preview) or Java 25+ (final)
- **Simple single-threaded scenarios** - Where thread pool reuse isn't a concern
- **Performance-critical code** - Where the overhead of scopes is unacceptable (rare)

Remember: Scoped Values are the future of thread-local storage in Java. They solve the fundamental problems with `ThreadLocal` in modern concurrent applications, especially in thread pool environments where data leakage is a serious concern!

