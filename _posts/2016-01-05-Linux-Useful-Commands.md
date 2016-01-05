---
layout: post
title: Linux Useful Commands
published: true
categories: 
            - Linux
---

#Motivation
Having worked mostly on windows based systems, I've converted to using Linux since the last 6 years. I've found some really useful commands
over the years so I thought I'd list them here.

###Event Designators !
The exclamation mark has several uses in Linux. I found two really useful ones.

####Re-execute command in history
We mostly use the up and down keys to scroll the the list of commands we have previously types.
If you use the *history* command it will give you a log of the commands you have previously executed. For example you may get the
following output:

 1988  ls
 1989  git status
 1990  git co
 1991  git co -b feature/test
 1992  git checkout -b feature/test
 1993  cd ../

If you wanted to re-execute command 1989 again, all you need to do is type `!1989` and it will perform the `git status` command again.

####Re-execute last command with sudo
Sometimes you execute a command and it fails because you forgot to prefix it with **sudo**. Sudo means run the command as superuser.
You could simply type `sudo !!` to execute the last command as superuser.


More information on event designators can be found [here](https://www.gnu.org/software/bash/manual/html_node/Event-Designators.html)
