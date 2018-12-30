---
layout: post
title: Streams and Futures
published: true
categories:
            - Java 8
            - Streams
            - Futures
---

# Motivation
Something that caught me out when streaming completable futures was the need to collect the futures before calling join on them otherwise the processing of the futures becomes serial and defeats the purpose.

## Original serial execution code
{% highlight java %}
List<String> words = asList("one", "two", "three", "four", "five");

List<Integer> lengths =
          words.stream()
                  .map(w -> CompletableFuture.supplyAsync(() -> w.length()))
                  .map(CompletableFuture::join)
                  .collect(toList());
{% endhighlight %}

The problem with the above code is that intermediate stream operations are lazy by nature. This means the above pipeline will only start to execute once .collect() is called i.e. it will execute in serial order.

## Modified pipeline to correct parallelism
You need to split the pipeline into two seperate pipelines with the first firing off the futures and the second completing them with join().

{% highlight java %}
List<String> words = asList("one", "two", "three", "four", "five");

List<Integer> pipeline1 =
          words.stream()
                  .map(w -> CompletableFuture.supplyAsync(() -> w.length()))
                  .collect(toList());
                  // simple collect() and do not join().

List<CompletableFuture<Integer>> pipeline2 =
                pipeline1.stream()
                        .map(CompletableFuture::join)
                        .collect(toList());
{% endhighlight %}
