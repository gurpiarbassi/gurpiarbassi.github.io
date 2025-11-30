---
layout: post
title: "AssertJ withRepresentation(): Better Error Messages with Custom Representations"
published: true
date: 2025-11-28
categories: java testing assertj
---

# Motivation

In this post, we'll explore AssertJ's **withRepresentation()** method - a powerful feature that allows you to customize how objects are displayed in error messages. This is especially useful when comparing complex objects like JSON, nested data structures, or custom classes where the default `toString()` representation doesn't provide clear, readable output.

## The Problem: Unreadable Error Messages

### Traditional Approach

```java
// Complex object with poor toString() implementation
class User {
    private String name;
    private String email;
    private Address address;
    private List<String> roles;
    
    // Default toString() - not very helpful
    @Override
    public String toString() {
        return "User@" + hashCode();
    }
}

// Test with unhelpful error message
@Test
void testUserComparison() {
    User actual = new User("John", "john@example.com", address, roles);
    User expected = new User("Jane", "jane@example.com", address, roles);
    
    assertThat(actual).isEqualTo(expected);
    // Error: expected: <User@123456> but was: <User@789012>
    // Not helpful at all!
}
```

### The Gap

- **Poor default toString()**: Many classes have unhelpful or missing toString() implementations
- **Complex nested structures**: Hard to see differences in nested objects
- **JSON/API responses**: Raw object representation doesn't show structure clearly
- **Missing**: A way to customize error message representation for better debugging

## What is withRepresentation()?

**withRepresentation()** is an AssertJ method that allows you to specify a custom representation function for objects in error messages. This makes it much easier to understand what's different when assertions fail, especially for complex objects like JSON, maps, or custom classes.

### Key Characteristics

- **Custom error messages**: Control how objects appear in failure messages
- **Better debugging**: See actual differences in a readable format
- **JSON-friendly**: Perfect for comparing JSON objects and API responses
- **Reusable**: Define representation once, use across multiple assertions
- **Flexible**: Works with any object type

## withRepresentation() API

### Basic Usage

```java
import org.assertj.core.api.Assertions;
import com.fasterxml.jackson.databind.ObjectMapper;

class BasicRepresentationExample {
    
    @Test
    void testWithCustomRepresentation() {
        User user = new User("John", "john@example.com");
        
        assertThat(user)
            .withRepresentation(u -> "User(name=" + u.getName() + 
                                   ", email=" + u.getEmail() + ")")
            .isNotNull();
    }
}
```

### JSON Representation

```java
import com.fasterxml.jackson.databind.ObjectMapper;

class JsonRepresentationExample {
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    @Test
    void testJsonObject() throws Exception {
        Map<String, Object> actual = Map.of(
            "name", "John",
            "age", 30,
            "email", "john@example.com"
        );
        
        Map<String, Object> expected = Map.of(
            "name", "Jane",
            "age", 30,
            "email", "jane@example.com"
        );
        
        assertThat(actual)
            .withRepresentation(obj -> {
                try {
                    return objectMapper.writerWithDefaultPrettyPrinter()
                        .writeValueAsString(obj);
                } catch (Exception e) {
                    return obj.toString();
                }
            })
            .isEqualTo(expected);
    }
}
```

### Ensuring Consistent JSON Ordering for Visual Comparison

**Important Note**: JSON objects are unordered by specification, and Java `Map` implementations (like `HashMap`) don't guarantee key order. This means when comparing JSON visually, keys might appear in different orders even if the content is identical, making it difficult to spot actual differences.

**Solution**: Sort keys before serialization to ensure consistent, deterministic output:

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import java.util.TreeMap;
import java.util.Map;
import java.util.LinkedHashMap;
import java.util.stream.Collectors;
import java.util.stream.StreamSupport;

class OrderedJsonRepresentationExample {
    private final ObjectMapper objectMapper = new ObjectMapper()
        .enable(SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS);
    
    @Test
    void testJsonObjectWithOrderedKeys() throws Exception {
        Map<String, Object> actual = Map.of(
            "name", "John",
            "age", 30,
            "email", "john@example.com"
        );
        
        Map<String, Object> expected = Map.of(
            "name", "Jane",
            "age", 30,
            "email", "jane@example.com"
        );
        
        assertThat(actual)
            .withRepresentation(this::toOrderedPrettyJson)
            .isEqualTo(expected);
    }
    
    private String toOrderedPrettyJson(Object obj) {
        try {
            Object ordered = sortKeysRecursively(obj);
            return objectMapper.writerWithDefaultPrettyPrinter()
                .writeValueAsString(ordered);
        } catch (Exception e) {
            return obj.toString();
        }
    }
    
    private Object sortKeysRecursively(Object obj) {
        if (obj instanceof Map) {
            @SuppressWarnings("unchecked")
            Map<String, Object> map = (Map<String, Object>) obj;
            return map.entrySet().stream()
                .sorted(Map.Entry.comparingByKey())
                .collect(Collectors.toMap(
                    Map.Entry::getKey,
                    e -> sortKeysRecursively(e.getValue()),
                    (e1, e2) -> e1,
                    LinkedHashMap::new
                ));
        } else if (obj instanceof Iterable && !(obj instanceof String)) {
            Iterable<?> iterable = (Iterable<?>) obj;
            return ((java.util.stream.Stream<?>) 
                java.util.stream.StreamSupport.stream(
                    iterable.spliterator(), false))
                .map(this::sortKeysRecursively)
                .collect(java.util.stream.Collectors.toList());
        }
        return obj;
    }
}
```

**Reusable Helper for Ordered JSON**

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.stream.Collectors;
import java.util.stream.StreamSupport;

class OrderedJsonHelper {
    private static final ObjectMapper objectMapper = new ObjectMapper();
    
    public static <T> Representation<T> orderedPrettyJson() {
        return obj -> {
            try {
                Object ordered = sortKeysRecursively(obj);
                return objectMapper.writerWithDefaultPrettyPrinter()
                    .writeValueAsString(ordered);
            } catch (Exception e) {
                return obj.toString();
            }
        };
    }
    
    @SuppressWarnings("unchecked")
    private static Object sortKeysRecursively(Object obj) {
        if (obj instanceof Map) {
            Map<String, Object> map = (Map<String, Object>) obj;
            return map.entrySet().stream()
                .sorted(Map.Entry.comparingByKey())
                .collect(Collectors.toMap(
                    Map.Entry::getKey,
                    e -> sortKeysRecursively(e.getValue()),
                    (e1, e2) -> e1,
                    LinkedHashMap::new
                ));
        } else if (obj instanceof Iterable && !(obj instanceof String)) {
            Iterable<?> iterable = (Iterable<?>) obj;
            return StreamSupport.stream(iterable.spliterator(), false)
                .map(OrderedJsonHelper::sortKeysRecursively)
                .collect(Collectors.toList());
        }
        return obj;
    }
}

// Usage
@Test
void testWithOrderedHelper() {
    Map<String, Object> data = Map.of("zebra", 1, "apple", 2, "banana", 3);
    
    assertThat(data)
        .withRepresentation(OrderedJsonHelper.orderedPrettyJson())
        .isNotNull();
    // Output will always have keys in alphabetical order:
    // {
    //   "apple" : 2,
    //   "banana" : 3,
    //   "zebra" : 1
    // }
}
```

## Real-World Use Cases

### 1. **JSON API Response Comparison**

```java
class ApiResponseTest {
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    @Test
    void testApiResponse() throws Exception {
        // Simulated API response
        String jsonResponse = """
            {
                "id": 123,
                "name": "John Doe",
                "email": "john@example.com",
                "address": {
                    "street": "123 Main St",
                    "city": "New York",
                    "zip": "10001"
                },
                "roles": ["user", "admin"]
            }
            """;
        
        Map<String, Object> actual = objectMapper.readValue(
            jsonResponse, new TypeReference<Map<String, Object>>() {});
        
        Map<String, Object> expected = Map.of(
            "id", 123,
            "name", "Jane Doe",  // Different name
            "email", "jane@example.com"
        );
        
        assertThat(actual)
            .withRepresentation(this::toPrettyJson)
            .containsAllEntriesOf(expected);
    }
    
    private String toPrettyJson(Object obj) {
        try {
            return objectMapper.writerWithDefaultPrettyPrinter()
                .writeValueAsString(obj);
        } catch (Exception e) {
            return obj.toString();
        }
    }
}
```

### 2. **Nested Object Comparison**

```java
class NestedObjectTest {
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    @Test
    void testNestedUserObject() throws Exception {
        User actual = new User(
            "John",
            "john@example.com",
            new Address("123 Main St", "New York", "10001"),
            List.of("user", "admin")
        );
        
        User expected = new User(
            "Jane",  // Different
            "jane@example.com",
            new Address("456 Oak Ave", "Boston", "02101"),  // Different
            List.of("user")  // Different
        );
        
        assertThat(actual)
            .withRepresentation(user -> {
                try {
                    Map<String, Object> map = Map.of(
                        "name", user.getName(),
                        "email", user.getEmail(),
                        "address", Map.of(
                            "street", user.getAddress().getStreet(),
                            "city", user.getAddress().getCity(),
                            "zip", user.getAddress().getZip()
                        ),
                        "roles", user.getRoles()
                    );
                    return objectMapper.writerWithDefaultPrettyPrinter()
                        .writeValueAsString(map);
                } catch (Exception e) {
                    return user.toString();
                }
            })
            .isEqualTo(expected);
    }
}
```

### 3. **List of Complex Objects**

```java
class ListComparisonTest {
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    @Test
    void testUserList() throws Exception {
        List<User> actual = List.of(
            new User("John", "john@example.com"),
            new User("Jane", "jane@example.com"),
            new User("Bob", "bob@example.com")
        );
        
        List<User> expected = List.of(
            new User("John", "john@example.com"),
            new User("Alice", "alice@example.com"),  // Different
            new User("Bob", "bob@example.com")
        );
        
        assertThat(actual)
            .withRepresentation(users -> {
                try {
                    List<Map<String, String>> userMaps = users.stream()
                        .map(u -> Map.of("name", u.getName(), "email", u.getEmail()))
                        .toList();
                    return objectMapper.writerWithDefaultPrettyPrinter()
                        .writeValueAsString(userMaps);
                } catch (Exception e) {
                    return users.toString();
                }
            })
            .isEqualTo(expected);
    }
}
```

### 4. **Map Comparison with JSON**

```java
class MapComparisonTest {
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    @Test
    void testConfigurationMap() throws Exception {
        Map<String, Object> actual = Map.of(
            "database", Map.of(
                "host", "localhost",
                "port", 5432,
                "name", "mydb"
            ),
            "cache", Map.of(
                "enabled", true,
                "ttl", 3600
            ),
            "features", List.of("feature1", "feature2", "feature3")
        );
        
        Map<String, Object> expected = Map.of(
            "database", Map.of(
                "host", "localhost",
                "port", 5432,
                "name", "production"  // Different
            ),
            "cache", Map.of(
                "enabled", true,
                "ttl", 7200  // Different
            ),
            "features", List.of("feature1", "feature2")  // Different
        );
        
        assertThat(actual)
            .withRepresentation(this::toPrettyJson)
            .isEqualTo(expected);
    }
    
    private String toPrettyJson(Object obj) {
        try {
            return objectMapper.writerWithDefaultPrettyPrinter()
                .writeValueAsString(obj);
        } catch (Exception e) {
            return obj.toString();
        }
    }
}
```

### 5. **Custom Class Representation**

```java
class CustomClassRepresentationTest {
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    @Test
    void testOrderObject() throws Exception {
        Order actual = new Order(
            "ORD-123",
            List.of(
                new OrderItem("Product A", 2, 29.99),
                new OrderItem("Product B", 1, 49.99)
            ),
            new Customer("John", "john@example.com")
        );
        
        Order expected = new Order(
            "ORD-123",
            List.of(
                new OrderItem("Product A", 3, 29.99),  // Different quantity
                new OrderItem("Product C", 1, 39.99)   // Different product
            ),
            new Customer("Jane", "jane@example.com")  // Different customer
        );
        
        assertThat(actual)
            .withRepresentation(order -> {
                try {
                    Map<String, Object> orderMap = Map.of(
                        "orderId", order.getOrderId(),
                        "items", order.getItems().stream()
                            .map(item -> Map.of(
                                "product", item.getProductName(),
                                "quantity", item.getQuantity(),
                                "price", item.getPrice()
                            ))
                            .toList(),
                        "customer", Map.of(
                            "name", order.getCustomer().getName(),
                            "email", order.getCustomer().getEmail()
                        )
                    );
                    return objectMapper.writerWithDefaultPrettyPrinter()
                        .writeValueAsString(orderMap);
                } catch (Exception e) {
                    return order.toString();
                }
            })
            .isEqualTo(expected);
    }
}
```

## Advanced Usage Patterns

### 1. **Reusable Representation Helper**

```java
class RepresentationHelper {
    private static final ObjectMapper objectMapper = new ObjectMapper();
    
    public static <T> Representation<T> jsonRepresentation() {
        return obj -> {
            try {
                return objectMapper.writerWithDefaultPrettyPrinter()
                    .writeValueAsString(obj);
            } catch (Exception e) {
                return obj.toString();
            }
        };
    }
    
    public static <T> Representation<T> compactJsonRepresentation() {
        return obj -> {
            try {
                return objectMapper.writeValueAsString(obj);
            } catch (Exception e) {
                return obj.toString();
            }
        };
    }
}

// Usage
@Test
void testWithHelper() {
    Map<String, Object> data = Map.of("key", "value");
    
    assertThat(data)
        .withRepresentation(RepresentationHelper.jsonRepresentation())
        .isNotNull();
}
```

### 2. **Conditional Representation**

```java
class ConditionalRepresentationTest {
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    @Test
    void testWithConditionalRepresentation() {
        Object data = getComplexObject();
        
        assertThat(data)
            .withRepresentation(obj -> {
                if (obj instanceof Map || obj instanceof List) {
                    try {
                        return objectMapper.writerWithDefaultPrettyPrinter()
                            .writeValueAsString(obj);
                    } catch (Exception e) {
                        return obj.toString();
                    }
                } else if (obj instanceof String) {
                    // Try to parse as JSON
                    try {
                        Object parsed = objectMapper.readValue((String) obj, Object.class);
                        return objectMapper.writerWithDefaultPrettyPrinter()
                            .writeValueAsString(parsed);
                    } catch (Exception e) {
                        return (String) obj;
                    }
                }
                return obj.toString();
            })
            .isNotNull();
    }
}
```

### 3. **Combining with Other Assertions**

```java
class CombinedAssertionsTest {
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    @Test
    void testMultipleAssertions() {
        Map<String, Object> response = getApiResponse();
        
        assertThat(response)
            .withRepresentation(this::toPrettyJson)
            .isNotNull()
            .containsKey("status")
            .containsEntry("status", "success")
            .extracting("data")
            .withRepresentation(this::toPrettyJson)
            .isNotNull();
    }
    
    private String toPrettyJson(Object obj) {
        try {
            return objectMapper.writerWithDefaultPrettyPrinter()
                .writeValueAsString(obj);
        } catch (Exception e) {
            return obj.toString();
        }
    }
}
```

### 4. **Extracting and Representing**

```java
class ExtractAndRepresentTest {
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    @Test
    void testExtractedField() {
        User user = new User("John", "john@example.com", address, roles);
        
        assertThat(user)
            .extracting("address")
            .withRepresentation(addr -> {
                try {
                    Address address = (Address) addr;
                    Map<String, String> addrMap = Map.of(
                        "street", address.getStreet(),
                        "city", address.getCity(),
                        "zip", address.getZip()
                    );
                    return objectMapper.writerWithDefaultPrettyPrinter()
                        .writeValueAsString(addrMap);
                } catch (Exception e) {
                    return addr.toString();
                }
            })
            .isNotNull();
    }
}
```

## Best Practices

### 1. **Create Reusable Representation Functions**

```java
// Good - reusable helper
class JsonAssertions {
    private static final ObjectMapper objectMapper = new ObjectMapper();
    
    public static <T> Representation<T> prettyJson() {
        return obj -> {
            try {
                return objectMapper.writerWithDefaultPrettyPrinter()
                    .writeValueAsString(obj);
            } catch (Exception e) {
                return obj.toString();
            }
        };
    }
}

// Usage
assertThat(data)
    .withRepresentation(JsonAssertions.prettyJson())
    .isEqualTo(expected);
```

### 2. **Handle Exceptions Gracefully**

```java
// Good - always returns a string, even on error
private String toPrettyJson(Object obj) {
    try {
        return objectMapper.writerWithDefaultPrettyPrinter()
            .writeValueAsString(obj);
    } catch (Exception e) {
        return "Error serializing to JSON: " + obj.toString();
    }
}
```

### 3. **Use for Complex Nested Structures**

```java
// Good - use for complex objects
assertThat(complexNestedObject)
    .withRepresentation(this::toPrettyJson)
    .isEqualTo(expected);

// Avoid - simple objects don't need it
assertThat(simpleString)
    .isEqualTo("expected");
// Don't overuse for simple comparisons
```

### 4. **Combine with Descriptive Assertions**

```java
// Good - combine with descriptive messages
assertThat(apiResponse)
    .withRepresentation(this::toPrettyJson)
    .as("API response should match expected structure")
    .isEqualTo(expected);
```

### 5. **Ensure Consistent JSON Key Ordering**

```java
// Important: JSON objects are unordered, so sort keys for consistent visual comparison
private String toOrderedPrettyJson(Object obj) {
    try {
        Object ordered = sortKeysRecursively(obj);
        return objectMapper.writerWithDefaultPrettyPrinter()
            .writeValueAsString(ordered);
    } catch (Exception e) {
        return obj.toString();
    }
}

// This ensures keys appear in the same order every time, making visual comparison easier
assertThat(actual)
    .withRepresentation(this::toOrderedPrettyJson)
    .isEqualTo(expected);
```

## Common Pitfalls

### 1. **Forgetting Exception Handling**

```java
// Bad - can throw exception
assertThat(data)
    .withRepresentation(obj -> 
        objectMapper.writerWithDefaultPrettyPrinter()
            .writeValueAsString(obj))  // Can throw!
    .isNotNull();

// Good - handle exceptions
assertThat(data)
    .withRepresentation(obj -> {
        try {
            return objectMapper.writerWithDefaultPrettyPrinter()
                .writeValueAsString(obj);
        } catch (Exception e) {
            return obj.toString();
        }
    })
    .isNotNull();
```

### 2. **Overusing for Simple Objects**

```java
// Bad - unnecessary for simple objects
assertThat("simple string")
    .withRepresentation(s -> s.toUpperCase())
    .isEqualTo("SIMPLE STRING");

// Good - only use for complex objects
assertThat(complexJsonObject)
    .withRepresentation(this::toPrettyJson)
    .isEqualTo(expected);
```

### 3. **Not Making Representations Reusable**

```java
// Bad - duplicated code
assertThat(data1)
    .withRepresentation(obj -> {
        try {
            return objectMapper.writerWithDefaultPrettyPrinter()
                .writeValueAsString(obj);
        } catch (Exception e) {
            return obj.toString();
        }
    })
    .isNotNull();

assertThat(data2)
    .withRepresentation(obj -> {
        try {
            return objectMapper.writerWithDefaultPrettyPrinter()
                .writeValueAsString(obj);
        } catch (Exception e) {
            return obj.toString();
        }
    })
    .isNotNull();

// Good - reusable helper
private String toPrettyJson(Object obj) {
    try {
        return objectMapper.writerWithDefaultPrettyPrinter()
            .writeValueAsString(obj);
    } catch (Exception e) {
        return obj.toString();
    }
}

assertThat(data1).withRepresentation(this::toPrettyJson).isNotNull();
assertThat(data2).withRepresentation(this::toPrettyJson).isNotNull();
```

### 4. **Ignoring JSON Key Ordering**

```java
// Bad - keys may appear in different orders, making comparison difficult
Map<String, Object> actual = Map.of("zebra", 1, "apple", 2, "banana", 3);
Map<String, Object> expected = Map.of("apple", 2, "banana", 3, "zebra", 1);

assertThat(actual)
    .withRepresentation(this::toPrettyJson)
    .isEqualTo(expected);
// Error message might show:
// Expected: {"apple":2,"banana":3,"zebra":1}
// Actual:   {"zebra":1,"apple":2,"banana":3}
// Hard to compare visually!

// Good - sort keys for consistent ordering
assertThat(actual)
    .withRepresentation(this::toOrderedPrettyJson)  // Keys sorted alphabetically
    .isEqualTo(expected);
// Error message will show:
// Expected: {"apple":2,"banana":3,"zebra":1}
// Actual:   {"apple":2,"banana":3,"zebra":1}
// Much easier to compare!
```

## Comparison: withRepresentation() vs Alternatives

| Approach | Use Case | Pros | Cons |
|----------|----------|------|------|
| **withRepresentation()** | Complex objects, JSON, nested structures | Clean, readable error messages | Requires representation function |
| **Custom toString()** | Simple objects | Built-in, automatic | Not flexible, affects all uses |
| **as() for messages** | Adding context | Simple | Doesn't change object representation |
| **Manual string building** | One-off cases | Full control | Verbose, error-prone |

## Summary

AssertJ's `withRepresentation()` method provides a powerful way to customize how objects appear in error messages. Key takeaways:

1. **Perfect for complex objects** - JSON, nested structures, custom classes
2. **Better debugging** - See actual differences in readable format
3. **JSON-friendly** - Excellent for API response comparison
4. **Reusable** - Create helper functions for common representations
5. **Exception-safe** - Always handle serialization errors gracefully

### When to Use withRepresentation()

- **JSON objects and API responses** - Make differences clearly visible
- **Complex nested structures** - See structure in error messages
- **Custom classes with poor toString()** - Provide better representation
- **Maps and Lists of complex objects** - Compare collections clearly
- **Debugging test failures** - Understand what's different quickly

### When Not to Use

- **Simple objects** - String, Integer, etc. don't need it
- **Objects with good toString()** - If default is already readable
- **Performance-critical tests** - Adds overhead to error message generation
- **One-time comparisons** - Consider if it's worth the setup

Remember: `withRepresentation()` is a powerful debugging tool. Use it to make your test failures more informative, especially when working with JSON, APIs, or complex data structures. The investment in creating good representations pays off when debugging test failures!

