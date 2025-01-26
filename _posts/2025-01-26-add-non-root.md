---
layout: post
title: Add a non root user for docker file
published: true
categories: 
            - Docker

---

# Motivation
It is best practice for your application to use a non root user for execution.

## Add user to a group
```
RUN addgroup -S spring && \
    adduser -S spring -G spring

```

## Switch to non-root user
```
USER spring:spring
```
