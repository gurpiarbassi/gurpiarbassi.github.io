---
layout: post
title: Java 8 and JPA 2.1 Attribute Converter
published: true
categories: 
            - Java 8
            - JPA
            - Attribute Converter

---

#Motivation
In JPA 2.1 there are several improvements to the specification. One of which is the use of attribute converters to convert betweeen Java types and database types.

JPA was released long before the new features in Java 8 such as the new java.time API. The JPA `@Temporal` annotation can only be applied to attributes of type java.util.Date and java.util.Calendar. If I wanted to use java.time objects in my entities I would need to explicitly tell JPA how I want the attributes converted otherwise JPA will map it to a BLOB instead.

##Example
In this example, I will show how to make use of the ZonedDateTime type in Java 8 with JPA 2.1.
{% highlight java linenos=table%}
@Converter(autoApply=true)
public class ZonedDateTimeConverter implements AttributeConverter<ZonedDateTime, Timestamp>{

    
    public Timestamp convertToDatabaseColumn(final ZonedDateTime entityValue) {
        return entityValue == null ? null : Timestamp.from(entityValue.toInstant());
    }

   
    public ZonedDateTime convertToEntityAttribute(final Timestamp databaseValue) {
        return (databaseValue == null ? null : ZonedDateTime.ofInstant(databaseValue.toInstant(), ZoneId.of("UTC")));
    }
}
{% endhighlight %}

Note that in the example we are explictly stating UTC as the timezone since it is good practice to store data as UTC in the database.

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>

<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
             http://xmlns.jcp.org/xml/ns/persistence/persistence_2_1.xsd"
             version="2.1">

    <persistence-unit name="xxxx" transaction-type="JTA">
        ...
        ...
        <!-- custom converters -->
        <class>com.gurps.ZonedDateTimeConverter</class>
        ...
        ...
    </persistence-unit>

</persistence>
{% endhighlight %}
