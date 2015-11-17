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

```
1.Map<String, List<String>> multimap = new HashMap<>();
..
..
2.public void addToMap(String key, String value){
3.  List<String> values = multimap.get(key);
4.  if(values == null){
5.    values = new ArrayList<>();
6.    multimap.put(key, values);
7.  }
8.  values.add(value);
9.}
```

###The Apache Commons approach
The package org.apache.commons.collections4 has a MultiMap class which implements java.util.Map. This can also be used to create a Map that has multiple values for a given key

```
 MultiMap mhm = new MultiValueMap();
 mhm.put(key, "A");
 mhm.put(key, "B");
 mhm.put(key, "C");
```

###The Java 8 lambda approach
However If you would like a pure Java approach without having to add dependancies such as commons-collections4 to your project, you can do the following:

```
10.public void addToMap(String key, String value){
11.  map.computeIfAbsent(key, k -> new ArrayList<>())
12.     .add(value);
13.}
```

###Explanation
Lines 3-6 are equivalent to line 11.
The computeIfAbsent method on the java.util.Map interface takes a key with which the specified value is to be associated and a mapping function used to compute the value. In our case the mapping function simply takes a key and create a new ArrayList.

Line 8 is equivalent to line 12.
