---
layout: post
title: Levenshtein Distance with StringUtils
published: true
categories: 
            - Apache Commons
            - String Utils
            - Levennshtein Distance
---

#Motivation
I've been using StringUtils from Apache Commons for a very long time. One of the most overlooked methods is 
*public static int getLevenshteinDistance(CharSequence s, CharSequence t).* You sometimes want to compare two strings and find out if they match closely enough or not. For example, this could be used in a search algorithm if a user keys in a word incorrectly e.g 'captiol'and we present them with a prompt saying 'Did you mean capital?'

##Theory behind the algorithm
I'm not going to attempt to explain the mathematics behind the algorithm. In a nutshell, the alorithm states:
*The Levenshtein distance between two words is the minimum number of single-character edits required to change one word into the other. It is named after Vladimir Levenshtein.*

##Implementation in Apache Commons
StringUtils provide two methods that compute the distance as an integer. The first method take two character sequences
*public static int getLevenshteinDistance(CharSequence s, CharSequence t)* and the other takes an additional threshold 
*public static int getLevenshteinDistance(CharSequence s, CharSequence t, int threshold)*

Lets look at a few examples of this at work:

{% highlight java %}
StringUtils.getLevenshteinDistance("fly", "ant") //returns 3
StringUtils.getLevenshteinDistance("elephant", "hippo") //return 7
StringUtils.getLevenshteinDistance("hello", "hallo") //returns 1
{% endhighlight %}
