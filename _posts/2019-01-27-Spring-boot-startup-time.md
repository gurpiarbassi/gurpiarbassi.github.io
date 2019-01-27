---
layout: post
title: Spring Boot slow to start
published: true
categories:
            - Java
            - Spring Boot
            - Networking
---

# Motivation
I was recently involved in a project where running unit tests of the SpringBoot flavour seemed to
be taking hours rather than seconds/minutes. After a big of digging around I found the answer on
stackoverlfow.com.

Application startup time was also extremley slow.

## Solution
I had a hunch that the problem was down to networking and localhost since the tests ran perfectly fine on our Jenkins build server and I could see the application startup quite fast on other peoples machines.

Following this stackoverflow post really helped https://stackoverflow.com/questions/33289695/inetaddress-getlocalhost-slow-to-run-30-seconds/33289897#33289897.

At this stage I do not know which operating systems / machines are affected by this issue.

The fix is pretty simple. You just need to add you hostname to the etc/hosts file.

I'm a Mac user so this is how I fixed it on my Macbook Pro 2014.

1. Find out your hostname by running the ```hostname``` command.

2. Edit your /etc/hosts file and add the hostname next to the mapping for 127.0.0.1 and ::1

e.g.

```
127.0.0.1   localhost myMacbookHostName.local
::1         localhost myMacbookHostName.local
```

My tests now take 5 minutes rather than 6 hours!
