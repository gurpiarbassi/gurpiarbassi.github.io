---
layout: post
title: Optional.orElse() is not lazy
published: true
categories: 
            - Java 8
            - Optional
---

# Motivation
On a recent project one of my team mates came accross a problem using Optional.orElse(T other). This method is not lazy so you need to be careful where you use it.

## The problem
You would think that Optional.orElse(T other) behaved similar to:

{% highlight java %}
if(x.isPresent()){
	return x.get();
}
else{
	return other;
}
{% endhighlight %}


However this is not the case. The 'other' value is evaluated immediately. Instead of simple values such as true/false, other can be a method call itself. Consider the following:
{% highlight java %}
if (x.isPresent()){
   getXFromSomeWhereElse();   
   return x.get()
} else {
   return getXFromSomeWhereElse();
}
{% endhighlight %}

In the above example, getXFromSomeWhereElse() is evaluated even though you don't need it. This can be problematic if getXFromSomeWhereElse() is a method that causes side effects e.g. database updates.

## The fix
For this type of scenario, you should use public T orElseGet(Supplier<? extends T> other).
The supplier will not be evaluated until it is needed.
{% highlight java %}
x.orElseGet( ()-> getXFromSomeWhereElse());
{% endhighlight %}