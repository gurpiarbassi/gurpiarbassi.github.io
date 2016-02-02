---
layout: post
title: Using the getent command on OS X
published: true
categories: 
            - Linux

---

##Motivation
I was recently given a shell script to run but it was not compatible with OS X. 
The script made use of a command called `getent`. The `getent` command basically performs a reverse DNS lookup i.e. given a host
name it will return the ip address.


##Creating a wrapper
In order to make this compatible with OS X, I had to create my own wrapper of the `getent` utility which delegates to an equivalent 
OS X command `dscacheutil`.

I created a file called getent. In the contents of this file I put:
```
#!/bin/bash
#assume arg 1 is passed in as 'hosts'
dscacheutil -q host -a name $2 | awk --field-separator=":" 'FNR == 2 {print $2}'
```

I then put this file into $HOME/bin. If the bin directory doesn't exist then you can create it.
Finally, go into your $HOME/.profile and add the line:
export SHELL_EXT=$HOME/bin

Then amend the line that exports the path to add $SHELL_EXT at the start (separator is colon)
export PATH="$SHELL_EXT:$PATH:.............



