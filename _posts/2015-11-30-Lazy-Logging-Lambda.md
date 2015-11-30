---
layout: post
title: The Lazy Logging Lambda
published: false
categories: 
            - Java 8
            - Multimap
            - Map
            - Log4j

---

#Motivation
We've all seen code at some point that looks like the following:


{% highlight java linenos=table%}
if(logger.isDebugEnabled()){  //by having this check we ensure that the string concatenation is not performed below.
  logger.debug("firstname =  " + firstName); //concat
}
{% endhighlight %}

This approach has some problems:
1) The fact that we have to make a call to `isDebugEnabled()` results in a extra line of code.
2) The statement has a guard condition to save on the computation spent on string concatenation. I've often made the mistake of 
changing the log statement to be `logger.info()` and leaving the guard condition to `if(logger.isDebugEnabled())`.

##How lamdafication can help

Java 8 Lambdas allow us to perform lazy evaluation. By using a lambda, the code above becomes:

{% highlight java linenos=table%}
  logger.debug(() -> "firstname =  " + firstName);
}
{% endhighlight %}
We don't need the guard condition since the string concatenation will only be done if 'logger.debug()' executes i.e. if log4j
configuration has debug enabled.

###The Apache Commons approach
The package org.apache.commons.collections4 has a MultiMap class which implements java.util.Map. This can also be used to create a Map that has multiple values for a given key

{% highlight java%}
 MultiMap mhm = new MultiValueMap();
 mhm.put(key, "A");
 mhm.put(key, "B");
 mhm.put(key, "C");
{% endhighlight %}

###The Java 8 lambda approach
However If you would like a pure Java approach without having to add dependencies such as commons-collections4 to your project, you can do the following:

{% highlight java linenos=table%}
public void addToMap(String key, String value){
  map.computeIfAbsent(key, k -> new ArrayList<>())
     .add(value);
}
{% endhighlight %}

###Explanation
Lines 5-8 are equivalent to line 13.
The `computeIfAbsent(..)` method on the java.util.Map interface takes a key with which the specified value is to be associated and a mapping function used to compute the value. In our case the mapping function simply takes a key and create a new ArrayList.

Line 10 is equivalent to line 14.
