---
layout: post
title: Google Drive Java API Through Http Proxy
published: true
categories: 
            - Java
            - Google Drive
---

#Motivation
I was recently working on a java project integrating the Google Drive Java API and came accross a problem when calls to Google
were not going through on a staging environment. I later realised that this was because all traffic on that environment needs to go through a 
HTTP proxy.

I then looked into setting the JVM arguments http.proxyHost and http.proxyPort. Normally this is all that is required. However, this would
not work for Google Drive.

##Solution

After debugging through the decompiled Google Drive code, I realised that the proxy needs to be set on the HttpTransport object programmatically.
The google code examples always show you the bit of code to get a HttpTransport object as follows:

`HttpTransport httpTransport = new NetHttpTransport();`

However, if you want to ensure proxy settings are picked up you need to do the following:

{% highlight java linenos=table%}

NetHttpTransport.Builder builder = new NetHttpTransport.Builder();
HttpTransport transport = builder.trustCertificates(GoogleUtils.getCertificateTrustStore())
                                 .setProxy(new Proxy(Proxy.Type.HTTP, new InetSocketAddress("myProxyHost", myProxyPort)))
                                 .build();

{% endhighlight %}


