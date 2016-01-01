I was recently using my work laptop and wanted to install some software on it using apt-get. This would normally not be a problem
if I was at work connected to the company network.

However this time I was at home and I kept getting timeouts. I later learnt that there was a proxy server configured which was preventing
me from installing anything through apt-get.
After looking around the web for answers I cam accross the following command:

```Acquire::http::proxy=false```

So all you need to do is ```sudo apt-get -O Acquire::http::proxy=false install <my-package-name>```
