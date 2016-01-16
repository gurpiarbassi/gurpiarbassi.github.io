---
layout: post
title: Reading Files with Java 8
published: true
categories: 
            - Java 8
            - File I/O
---

#Motivation
The latest features in Java 8 have proved to be very useful when reading files.

##The newBufferedReader() static method on the java.nio.file.Files class.
{% highlight java %}
try (BufferedReader br = Files.newBufferedReader(Paths.get(fileName)))
{% endhighlight %}

##The lines() method on the BufferedReader class

{% highlight java linenos=table%}
final String fileName = "/tmp/data.txt";
final List<String> list = new ArrayList<>();
try (BufferedReader br = Files.newBufferedReader(Paths.get(fileName))){
			list = br.lines().collect(Collectors.toList());
}
{% endhighlight %}


##The lines() method on the Files class

{% highlight java %}
final String fileName = "/tmp/data.txt";
try (Stream<String> stream = Files.lines(Paths.get(fileName))) 
{% endhighlight %}

##Skipping lines with BufferedReader
{% highlight java %}
final List<String> list = new ArrayList<>();
try (BufferedReader br = Files.newBufferedReader(Paths.get(fileName))) {
			list = br.lines()
			         .skip(5) // skip the first 5 lines
			         .collect(Collectors.toList());
}

{% endhighlight %}

