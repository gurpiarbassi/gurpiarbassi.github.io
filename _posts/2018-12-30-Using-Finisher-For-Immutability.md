---
layout: post
title: Finisher function
published: true
categories:
            - Java 8
            - Streams
            - Collectors
            - Immutable
---

# Motivation
After using a collector such as toList() or toSet(), the collection you end up with is not immutable. We need to find a way to
cleanly wrap this collection into a UnmodifiableList or UnmodifiableSet.

## Solution
We can make use the Collectors.collectingAndThen method that takes two arguments: a downstream collector and a finisher function.
{% highlight java %}
Arrays.stream(elements)
      .collect(collectingAndThen(toList(), Collections::UnmodifiableList));
{% endhighlight %}
