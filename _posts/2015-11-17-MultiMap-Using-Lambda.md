---
layout: post
title: The classic multi-map idiom using Java 8 Lambda Calculus
published: true
categories: 
            - Java 8
            - Multimap
            - Map

---

#Motivation
Often you see yourself and other writing a code using a multimap idiom to store more than one value against a given key in a Map data structure. With the introduction of Lambda Calculus in Java 8 it has become a lot easier to write such idioms.

###The classic approach

{% highlight java linenos=table%}
Map<String, List<String>> multimap = new HashMap<>();
..
..
public void addToMap(String key, String value){
  List<String> values = multimap.get(key);
  if(values == null){
    values = new ArrayList<>();
    multimap.put(key, values);
  }
  values.add(value);
}
{% endhighlight %}


###The Apache Commons approach
The package org.apache.commons.collections4 has a MultiMap class which implements java.util.Map. This can also be used to create a Map that has multiple values for a given key

{% highlight java %}
 MultiMap mhm = new MultiValueMap();
 mhm.put(key, "A");
 mhm.put(key, "B");
 mhm.put(key, "C");
{% endhighlight %}

###The Java 8 lambda approach
However If you would like a pure Java approach without having to add dependencies such as commons-collections4 to your project, you can do the following:

{% highlight java %}
public void addToMap(String key, String value){
  map.computeIfAbsent(key, k -> new ArrayList<>())
     .add(value);
}
{% endhighlight %}

###Explanation
Lines 5-8 are equivalent to line 11.
The `computeIfAbsent(..)` method on the java.util.Map interface takes a key with which the specified value is to be associated and a mapping function used to compute the value. In our case the mapping function simply takes a key and create a new ArrayList.

Line 8 is equivalent to line 12.
