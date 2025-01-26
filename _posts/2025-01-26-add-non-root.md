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
Creates a group called spring (using -S), created a system user called spring (using -S) and adds to the group spring (using -G)
```
RUN addgroup -S spring && \
    adduser -S spring -G spring

```

## Switch to non-root user
```
USER spring:spring
```
