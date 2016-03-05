---
layout: post
title: The Lazy Logging Lambda
published: true
categories: 
            - Java 8
            - Multimap
            - Map
            - Log4j

---

# Motivation
We've all seen code at some point that looks like the following:


{% highlight java linenos=table%}
if(logger.isDebugEnabled()){  //by having this check we ensure that the string concatenation is not performed below.
  logger.debug("firstname =  " + firstName); //concat
  }
{% endhighlight %}

1. The fact that we have to make a call to `isDebugEnabled()` results in a extra line of code.
2. The statement has a guard condition to save on the computation spent on string concatenation. I've often made the mistake of changing the log statement to be `logger.info()` and leaving the guard condition to `if(logger.isDebugEnabled())`.

## How lamdafication can help

Java 8 Lambdas allow us to perform lazy evaluation. By using a lambda, the code above becomes:

{% highlight java linenos=table%}
  logger.debug(() -> "firstname =  " + firstName);
{% endhighlight %}
We don't need the guard condition since the string concatenation will only be done if 'logger.debug()' executes i.e. if log4j configuration has debug enabled. The log methods have been extended to take a functional interface (supplier) to construct the log statement.

By passing a functional interface (lambda) we are effectivly passing behaviour around and it is executed only when required by the method using it.

Note that this also assumes you are using log4j2 since it comes with Java 8 Lambda support.

