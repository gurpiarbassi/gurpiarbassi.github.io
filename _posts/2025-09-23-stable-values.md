---
layout: post
title: "Java 25 Stable Values: Bridging the Gap Between Final and Non-Final Fields"
published: true
date: 2025-09-23
categories: java programming
---

# Motivation

In this post, we'll explore Java 25's new **Stable Values API** - a powerful feature that fills the gap between `final` fields (which can only be assigned once) and non-final fields (which can be reassigned multiple times). This API provides a middle ground for scenarios where you need controlled, one-time initialization with better safety guarantees.

## The Problem: Final vs Non-Final Fields

### Traditional Approach

```java
public class TraditionalExample {
    // Final field - can only be assigned once, but must be assigned at declaration or in constructor
    private final String name;
    
    // Non-final field - can be reassigned multiple times, but lacks safety guarantees
    private String description;
    
    public TraditionalExample(String name) {
        this.name = name; // OK - final field assignment in constructor
        this.description = "Default"; // OK - but can be changed later
    }
    
    public void updateDescription(String newDescription) {
        this.description = newDescription; // OK - but no safety guarantees
    }
}
```

### The Gap

- **Final fields**: Safe but inflexible - must be assigned at declaration or in constructor
- **Non-final fields**: Flexible but unsafe - can be reassigned anywhere, anytime
- **Missing**: A way to assign a value once after construction with safety guarantees

## What are Stable Values?

**Stable Values** provide a controlled way to initialize fields after object construction while maintaining safety guarantees. Once a stable value is set, it cannot be changed, similar to final fields, but the assignment can happen after construction.

### Key Characteristics

- **One-time assignment**: Can only be set once after construction
- **Thread-safe**: Safe for concurrent access
- **Null-safe**: Prevents accidental null assignments
- **Immutable**: Once set, the value cannot be changed

## Stable Values API

### Basic Usage

```java
import java.util.StableValue;

public class StableValueExample {
    private final StableValue<String> name = StableValue.empty();
    private final StableValue<Integer> age = StableValue.empty();
    private final StableValue<Address> address = StableValue.empty();
    
    public void initializeUser(String name, int age, Address address) {
        // Set values using .of() method
        this.name.of(name);
        this.age.of(age);
        this.address.of(address);
    }
    
    public String getName() {
        return name.get(); // Throws IllegalStateException if not set
    }
    
    public Integer getAge() {
        return age.orElse(0); // Returns default value if not set
    }
}
```

### The `.of()` Method

The `.of()` method sets the stable value if it hasn't been set yet:

```java
public class OfMethodExample {
    private final StableValue<String> config = StableValue.empty();
    
    public void loadConfiguration() {
        // First call - sets the value
        config.of("production");
        
        // Second call - throws IllegalStateException
        // config.of("development"); // This would fail!
    }
    
    public String getConfig() {
        return config.get(); // Returns "production"
    }
}
```

### The `.orElseSet()` Method

The `.orElseSet()` method sets the value only if it hasn't been set yet, otherwise returns the current value:

```java
public class OrElseSetExample {
    private final StableValue<String> databaseUrl = StableValue.empty();
    
    public void initializeDatabase() {
        // Try to set from environment variable
        String envUrl = System.getenv("DATABASE_URL");
        if (envUrl != null) {
            databaseUrl.orElseSet(envUrl);
        }
        
        // Fallback to default - only sets if not already set
        databaseUrl.orElseSet("jdbc:postgresql://localhost:5432/mydb");
    }
    
    public String getDatabaseUrl() {
        return databaseUrl.get();
    }
}
```

## Real-World Use Cases

### 1. **Lazy Initialization with Safety**

```java
public class LazyInitializedService {
    private final StableValue<DatabaseConnection> connection = StableValue.empty();
    private final StableValue<CacheManager> cache = StableValue.empty();
    
    public DatabaseConnection getConnection() {
        if (!connection.isSet()) {
            connection.of(createDatabaseConnection());
        }
        return connection.get();
    }
    
    public CacheManager getCache() {
        cache.orElseSet(createCacheManager());
        return cache.get();
    }
    
    private DatabaseConnection createDatabaseConnection() {
        // Expensive database connection creation
        return new DatabaseConnection();
    }
    
    private CacheManager createCacheManager() {
        // Cache initialization
        return new CacheManager();
    }
}
```

### 2. **Configuration Management**

```java
public class ApplicationConfig {
    private final StableValue<String> apiKey = StableValue.empty();
    private final StableValue<Integer> port = StableValue.empty();
    private final StableValue<Boolean> debugMode = StableValue.empty();
    
    public void loadFromFile(String configFile) {
        Properties props = loadProperties(configFile);
        
        // Set values only if they exist in config file
        if (props.containsKey("api.key")) {
            apiKey.of(props.getProperty("api.key"));
        }
        
        if (props.containsKey("server.port")) {
            port.of(Integer.parseInt(props.getProperty("server.port")));
        }
        
        if (props.containsKey("debug")) {
            debugMode.of(Boolean.parseBoolean(props.getProperty("debug")));
        }
    }
    
    public void setDefaults() {
        // Set default values only if not already set
        apiKey.orElseSet("default-api-key");
        port.orElseSet(8080);
        debugMode.orElseSet(false);
    }
    
    public String getApiKey() {
        return apiKey.orElseThrow(() -> new IllegalStateException("API key not configured"));
    }
    
    public int getPort() {
        return port.orElse(8080);
    }
    
    public boolean isDebugMode() {
        return debugMode.orElse(false);
    }
}
```

### 3. **Builder Pattern with Stable Values**

```java
public class UserBuilder {
    private final StableValue<String> name = StableValue.empty();
    private final StableValue<String> email = StableValue.empty();
    private final StableValue<Integer> age = StableValue.empty();
    
    public UserBuilder setName(String name) {
        this.name.of(name);
        return this;
    }
    
    public UserBuilder setEmail(String email) {
        this.email.of(email);
        return this;
    }
    
    public UserBuilder setAge(int age) {
        this.age.of(age);
        return this;
    }
    
    public User build() {
        // Ensure required fields are set
        if (!name.isSet()) {
            throw new IllegalStateException("Name is required");
        }
        
        return new User(
            name.get(),
            email.orElse(""),
            age.orElse(0)
        );
    }
}
```

## Comparison: Final vs Non-Final vs Stable Values

| Feature | Final Fields | Non-Final Fields | Stable Values |
|---------|--------------|------------------|---------------|
| **Assignment Time** | Declaration or constructor | Anytime | After construction |
| **Reassignment** | Never | Multiple times | Never (once set) |
| **Thread Safety** | Yes | No | Yes |
| **Null Safety** | No | No | Yes |
| **Lazy Initialization** | No | Yes | Yes |
| **Safety Guarantees** | High | Low | High |

## Advanced Usage Patterns

### 1. **Conditional Initialization**

```java
public class ConditionalInitialization {
    private final StableValue<String> environment = StableValue.empty();
    
    public void detectEnvironment() {
        // Try different detection methods
        if (System.getenv("ENV") != null) {
            environment.orElseSet(System.getenv("ENV"));
        } else if (System.getProperty("environment") != null) {
            environment.orElseSet(System.getProperty("environment"));
        } else {
            environment.orElseSet("development");
        }
    }
    
    public String getEnvironment() {
        return environment.get();
    }
}
```

### 2. **Resource Management**

```java
public class ResourceManager {
    private final StableValue<ExecutorService> executor = StableValue.empty();
    private final StableValue<ScheduledExecutorService> scheduler = StableValue.empty();
    
    public ExecutorService getExecutor() {
        executor.orElseSet(Executors.newFixedThreadPool(10));
        return executor.get();
    }
    
    public ScheduledExecutorService getScheduler() {
        scheduler.orElseSet(Executors.newScheduledThreadPool(5));
        return scheduler.get();
    }
    
    public void shutdown() {
        if (executor.isSet()) {
            executor.get().shutdown();
        }
        if (scheduler.isSet()) {
            scheduler.get().shutdown();
        }
    }
}
```

### 3. **Validation and Transformation**

```java
public class ValidatedStableValue {
    private final StableValue<String> email = StableValue.empty();
    
    public void setEmail(String email) {
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("Invalid email format");
        }
        this.email.of(email.toLowerCase().trim());
    }
    
    public String getEmail() {
        return email.orElseThrow(() -> new IllegalStateException("Email not set"));
    }
}
```

## Best Practices

### 1. **Always Use `.orElseSet()` for Optional Values**

```java
// Good - provides fallback
config.orElseSet("default-value");

// Avoid - throws exception if not set
// config.get(); // This will throw if not set
```

### 2. **Use `.of()` for Required Values**

```java
// Good - makes it clear this is required
public void setRequiredValue(String value) {
    this.requiredValue.of(value);
}
```

### 3. **Check State Before Operations**

```java
public void performOperation() {
    if (!config.isSet()) {
        throw new IllegalStateException("Configuration not initialized");
    }
    // Perform operation with config.get()
}
```

### 4. **Use in Immutable Objects**

```java
public class ImmutableUser {
    private final StableValue<String> name = StableValue.empty();
    private final StableValue<String> email = StableValue.empty();
    
    public ImmutableUser(String name, String email) {
        this.name.of(name);
        this.email.of(email);
    }
    
    // No setters - values cannot be changed after construction
    public String getName() { return name.get(); }
    public String getEmail() { return email.get(); }
}
```

## Common Pitfalls

### 1. **Forgetting to Set Values**

```java
// Bad - will throw IllegalStateException
public String getValue() {
    return stableValue.get(); // Throws if not set
}

// Good - provides fallback
public String getValue() {
    return stableValue.orElse("default");
}
```

### 2. **Trying to Set Multiple Times**

```java
// Bad - second call will throw
stableValue.of("first");
stableValue.of("second"); // IllegalStateException!

// Good - use orElseSet for conditional setting
stableValue.orElseSet("first");
stableValue.orElseSet("second"); // Ignored, keeps "first"
```

### 3. **Not Handling Null Values**

```java
// Bad - can set null
stableValue.of(null);

// Good - validate before setting
if (value != null) {
    stableValue.of(value);
}
```

## Summary

Java 25's Stable Values API provides an elegant solution for controlled, one-time initialization of fields after object construction. Key takeaways:

1. **Bridges the gap** between final and non-final fields
2. **Thread-safe** and **null-safe** by design
3. **One-time assignment** with safety guarantees
4. **Perfect for lazy initialization** and configuration management
5. **Use `.of()` for required values**, `.orElseSet()` for optional values

### When to Use Stable Values

- **Lazy initialization** of expensive resources
- **Configuration management** with fallbacks
- **Builder patterns** with validation
- **Immutable objects** with post-construction initialization
- **Resource management** with controlled lifecycle

### When to Stick with Traditional Approaches

- **Simple final fields** for values known at construction time
- **Regular fields** when you need multiple reassignments
- **Performance-critical** code where the overhead is not acceptable

Remember: Stable Values are a powerful tool, but use them judiciously. They're perfect for scenarios where you need the safety of final fields with the flexibility of post-construction initialization!
