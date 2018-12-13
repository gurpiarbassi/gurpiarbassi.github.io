---
layout: post
title: Stream concatenation
published: true
categories:
            - Java 8
            - Streams
---

# Motivation
Sometime you have to concatenate multiple streams together before you collect them.
Calling the Stream::concat method multiple times isn't very readable.

## Three ways to do it

{% highlight java linenos=table%}
public class Tests {


    @Test
    public void streamConcatUsingConcat() {
        Stream<String> first = Stream.of("a", "b", "c");
        Stream<String> second = Stream.of("d", "e", "f");
        Stream<String> third = Stream.of("g", "h", "i");


        // Potential stack overflow can occur if the call chain is deep with concat() usage

        // Java docs say:
        // Use caution when constructing streams from repeated concatenation.
        // Accessing an element of a deeply concatenated stream can result in deep call chains, or even StackOverflowError.
        final Stream<String> finalStream = Stream.concat(Stream.concat(first, second), third);

        final List<String> result = finalStream.collect(toList());

        assertThat(result, is(Arrays.asList("a", "b", "c", "d", "e", "f", "g", "h", "i")));
    }

    @Test
    public void streamConcatUsingReduce() {
        Stream<String> first = Stream.of("a", "b", "c");
        Stream<String> second = Stream.of("d", "e", "f");
        Stream<String> third = Stream.of("g", "h", "i");


        // Potential stack overflow can occur if the call chain is deep with concat() usage:

        // Java docs say:
        // Use caution when constructing streams from repeated concatenation.
        // Accessing an element of a deeply concatenated stream can result in deep call chains, or even StackOverflowError.
        final Stream<String> finalStream = Stream.of(first, second, third).reduce(Stream.empty(), Stream::concat);


        final List<String> result = finalStream.collect(toList());

        assertThat(result, is(Arrays.asList("a", "b", "c", "d", "e", "f", "g", "h", "i")));

    }

    @Test
    public void streamConcatUsingFlatMap() {
        Stream<String> first = Stream.of("a", "b", "c");
        Stream<String> second = Stream.of("d", "e", "f");
        Stream<String> third = Stream.of("g", "h", "i");

        // Only quirk/problem with using flatMap is that it forces its input streams into sequential mode even if they
        // were originally parallel.

        // The outermost concatenated stream can still be made parallel, and we will be able to process elements from
        // distinct input streams in parallel but the elements of each individual input stream must all be processed
        // sequentially.

        final Stream<String> finalStream = Stream.of(first, second, third).flatMap(Function.identity());


        final List<String> result = finalStream.collect(toList());

        assertThat(result, is(Arrays.asList("a", "b", "c", "d", "e", "f", "g", "h", "i")));

        // An even better solution is to use a Spliterator like in:
        // https://github.com/TechEmpower/misc/blob/master/concat/src/main/java/rnd/StreamConcatenation.java

    }
{% endhighlight %}
