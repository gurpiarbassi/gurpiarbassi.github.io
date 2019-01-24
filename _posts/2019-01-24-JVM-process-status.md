---
layout: post
title: Java process status reporting
published: true
categories:
            - Java
            - jps
---

# Discovery
I found a handy little utility called jps that ships with Java to give you the status of java processes running on your machine.

## Solution
```
 jps
```
That command will list all the java processes running.

It comes with the following flags too:

```
-q
Suppress the output of the class name, JAR file name, and arguments passed to the main method.

-m
Output the arguments passed to the main method.

-l
Output the full package name for the application's main class or the full path name to the application's JAR file.

-v
Output the arguments passed to the JVM.

-V
Output the arguments passed to the JVM through the flags file (the .hotspotrc file or the file specified by the -XX:Flags=<filename> argument).
```

Sample Output

16311 net.sourceforge.squirrel_sql.client.Main --log-config-file /Applications/SQuirreLSQL.app/Contents/Resources/Java/log4j.properties --squirrel-home /Applications/SQuirreLSQL.app/Contents/Resources/Java --native-laf
