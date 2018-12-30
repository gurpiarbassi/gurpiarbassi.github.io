---
layout: post
title: Downstream Collectors
published: true
categories:
            - Java 8
            - Streams
            - Collectors
---

# Motivation
To postprocess the collection of objects produced by a upstream operation such as partitioning or grouping.
In this example we will partition a collection of strings by odd or even length and then count how many in
each partition.

## Solution
{% highlight java %}
List<String> words = asList("one", "two", "three", "four", "five");

Map<Boolean, Long> strings =
          words.stream()
                  .collect(partitioningBy(s -> s.length() % 2 == 0, Collectors.counting()));
{% endhighlight %}

The Collectors.counting() method is a downstream collector because it performs postprocessing after the partitioning is complete. You end up with a map like:
true: 2
false: 3
