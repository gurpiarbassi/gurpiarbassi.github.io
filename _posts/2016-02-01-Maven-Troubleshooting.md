---
layout: post
title: JRE Issues with Maven
published: true
categories: 
            - Java
            - Maven

---

#Motivation
I recently faced a problem where I had updated my Java installation with some extended cryptography jars. However when I ran
`clean install` from maven, the build failed with an error indicating that these jars had made no difference.

#Running maven with Debug
You can run maven with the -X flag or --debug flag.
So I ran the command `mvn clean install | less` to pipe the output to less. This helped me to realise that JAVA_HOME it was using was pointing to another JRE and not the one I dropped the cryptography jars into.
