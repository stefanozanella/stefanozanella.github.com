---
layout: post
title: "Speaking in tongues with SSL"
date: 2013-01-22 16:56
comments: true
categories: [devops, tricks, ssl]
---

This was a nice one; let me explain.

I've setup a PotgreSQL host to hold all the various databases for the various
services deployed in the infrastructure. Since I know I'm not a security
expert, where I can I try to do the bare minumum needed and use SSL. This is
the case for PostgreSQL; not only, but I try hard to enforce two-way
certificate validation where possibile.

<!-- More -->
It turns out that despite the simplicity behind the concept of two-way
certificate validation, very few _"modern"_ services support that in a
user-friendly way. I already had a chance to rant a bit on Twitter about the
problems Puppet is currently facing with SSL; this time I want to tell you this
story that involves the [Gerrit Code Review](https://code.google.com/p/gerrit/)
application and the way I solved the problem, which in my opinion is quite
hacky (and will possibly break things in a near future).

First, a little overview of two-way SSL certificate validation. Basically, in
normal circumstances, during a SSL handshake, only the client verifies that the
certificate the server is providing is valid (checking against a list of known
and trusted Certificate Authorities); when performing two-way validation, this
process is true also for the server. That is, the server expects the client to
send a certificate and verifies that it can be trusted with the same mechanism.  
This is useful when you're running your own PKI and can freely issue
certificates to all your hosts; it becomes a form of authentication similar to
that we all use when doing public key authentication in SSH (maybe even
stronger).  
For this to work, obviously, there must be explicit configuration support on 
the client service to point to certificate, private key and CA that will be
used when handshaking with the server. Also, note that you cannot simply
disable certificate validation, since this will only lower the barrier **on one
side** of the communication channel: the server will still require you (the
client) to provide a valid certificate (in most cases you can tell the server
to relax the constraint, but then this discussion would become a little
pointless).

Here is where problems begin: let's look specifically at how this can be
handled in Gerrit.  
In particular, let's see how a typical database setup looks like in Gerrit:
``` ini
[database]
  type = POSTGRESQL
  hostname = postgresql.derecom.it
  database = gerrit
  username = gerrit
```
This will tell JDBC to connect to `postgresql.derecom.it` and look for database
`gerrit`, authenticating with user `gerrit` (password is handled in another
file). See? No mention to **SSL**. Unofrtunately, JDBC doesn't automatically
recognizes that it needs to setup a SSL connection, and trying to boot the
gerrit server results in this kind of error on the PosgtreSQL host:
```
FATAL:  no pg_hba.conf entry for host "x.y.z.w", user "gerrit", database "gerrit", SSL off
```
(that's because I **don't allow unencrypted connections to databases**).

It seems that there's no hope to solve this. Luckily, though, Gerrit use JDBC
under the hood to manage the connection pool; looking through 
[JDBC PosgtreSQL driver documentation](http://jdbc.postgresql.org/documentation/80/connect.html),
we can read that:
>  In addition to the standard connection parameters the driver supports a
>  number of additional properties which can be used to specify additional
>  driver behavior specific to PostgreSQL™. These properties may be specified
>  in either the connection URL or an additional Properties object parameter to
>  DriverManager.getConnection. The following examples illustrate the use of
>  both methods to establish a SSL connection.
> 
> String url = "jdbc:postgresql://localhost/test";  
> Properties props = new Properties();  
> props.setProperty("user","fred");  
> props.setProperty("password","secret");  
> props.setProperty("ssl","true");  
> Connection conn = DriverManager.getConnection(url, props);  
> 
> String url =  
> "jdbc:postgresql://localhost/test?user=fred&password=secret**&ssl=true**";  
> Connection conn = DriverManager.getConnection(url);  

So, it seems that if we could pass a `ssl` parameter to the JDBC URL we could
enable SSL while connection to PostgreSQL. A first solution is to set the
connection type to `JDBC` instead of `POSTGRESQL` in Gerrit configuration. This
would allow you to directly specify the URL JDBC should connect to, parameters
included.  
Since I didn't know if I would have had to specify also the user/pass in that
same URL, I wanted to try to stick to the `POSTGRESQL` type. Looking at 
[this class' source code](https://gerrit.googlesource.com/gerrit/+/7029fc15df86e6ef886d67a8117a39d21320fe60/gerrit-pgm/src/main/java/com/google/gerrit/pgm/util/DataSourceProvider.java),
it turns out that Gerrit itself builds a JDBC URL, so using one of the other
connection types is just a tiny wrapper over the JDBC type. However, the URL is
built leaving out username and password, which are set separately when actually
instantiating the connection, and the last variable in the URL concatenation is
the database name. So, I modified the configuration by directly appending the
`ssl` param to the database name:
``` ini
[database]
  type = POSTGRESQL
  hostname = postgresql.derecom.it
  database = gerrit?ssl=true
  username = gerrit
```

You know what? That works like a charm :)  
Obviously this is a bit hacky, and surely setting a JDBC is a more proper way
to handle this thing, however if it works...

**PS:** I left out from the discussion a fundamental step, which is passing the
certificate, private key and CA to Gerrit to correctly handle the SSL
handshake. This involves passing two additional properties to Gerrit startup
command; I've already written down about it in 
[this Gist](https://gist.github.com/4124338), so I won't repeat myself here.

## Conclusion
So, having spent almost 2 hours fixing this issue and writing about it, what I
can hope for is that developers start to care a little more about SSL and the
various ways it can be used. This would surely help changing the impression
that SSL is a though beast, as also noticed by
[Brice Figureau](https://twitter.com/_masterzen_) in 
[this post about Puppet SSL PKI](http://www.masterzen.fr/2010/11/14/puppet-ssl-explained/), and making our
infrastructures more secure overall.
