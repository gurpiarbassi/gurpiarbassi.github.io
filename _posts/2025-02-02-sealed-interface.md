---
layout: post
title: Using Sealed Interfaces and Records to Consume REST API
published: true
categories: 
            - Sealed Interfaces
            - Records

---

# Selead interfaces and records to consume REST API

In this example we will consume a REST API and use records to represent the result. The result could be a success of a failure.

```java
public class ProductRecords {
    public record Product(String name, BigDecimal price) {}

    public record ProductResponse (List<Product> products) {}

    public sealed interface Result {
        record Success(ProductResponse response) implements Result {}
        record Error(String message) implements Result {}
    }
}
```

In the client to the REST API we can use the `Result` interface to handle the response.


I want to use the `Result` interface to handle the response. When there is an error in the API I want to invoke Result.Error and on success I want to invoke Result.Success. This can be done interrogating the status code of the response as follows:

```java
if (response.getStatusCode().is2xxSuccessful()) {
    return new Result.Success(gson.fromJson(response.getBody(), ProductResponse.class));
} else {
    return new Result.Error("error" + response.statusCode());
}
``` 

This then allows me to use a switch statement without having a default case since I have covered all the cases with the sealed interface.

```java
switch (result) {
    case Result.Success success -> {
        // handle success
    }
    case Result.Error error -> {
        // handle error
    }
}
```







