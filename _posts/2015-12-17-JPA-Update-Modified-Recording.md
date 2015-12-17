---
layout: post
title: Recording last updated and created timestamp on a JPA entity
published: true
categories: 
            - Java
            - JPA

---

#Motivation
It is a common use case for an application to audit the date/time when a database record was first inserted and the date/time when it was subsequently
modified. There are several ways to do this ranging from database triggers to programmatically controlling the timestamps through code.

JPA has a nice feature which allows you to define callback methods that respond to an entities lifecycle events.
Consider the following example:

{% highlight java linenos=table%}
public class Customer{
  private String firstname;
  private String lastname;
  private ZonedDateTime createdTimestamp;
  private ZonedDateTime updatedTimestamp;
  
  @PrePersist
  void onCreate(){
    createdTimestamp = ZonedDateTime.now(ZoneId.of("UTC"));
    updatedTimestamp = createdTimestamp;
  }
  
  @PreUpdate
  void onUpdate(){
    updatedTimestamp = ZonedDateTime.now(ZoneId.of("UTC"));
  }
}
{% endhighlight %}

1. The `@PrePersist` annotation responds to the persist lifecycle event by intialising the createdTimestamp to the current date and time.
It also ensures the updatedTimestamp is persisted with the same value just to keep things tidy.

2. The `@PreUpdate` annotation gets called when the entity is about to be updated. Here we simply ensure we have the current date/time again.
