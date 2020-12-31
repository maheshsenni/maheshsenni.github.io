---
title: "Host Name Resolution in Java"
date: 2020-01-26T19:19:46-06:00
draft: false
---

It is very rare nowadays to find code that uses IP address to connect to servers — especially with the popularity of cloud based infrastructure and microservices. It is convenient to have human-friendly host names than a few random numbers put together. “mydatabase.company.com” is much easier to comprehend than “10.202.17.5”. It wouldn’t be an understatement if I said, host name resolution is one of the most overlooked and taken for granted functionality in software applications.

Most often than not if you are thinking about DNS resolution, it is highly possible that you are facing connectivity issues and your application is not working as expected. Well, I was in the exact situation recently — we migrated our database to a new server and the [CNAME](https://en.wikipedia.org/wiki/CNAME_record) was updated to point to the new IP address. One of our Spring Boot applications was still writing to the old server even after the CNAME was switched to the new server. Although resolving the problem was straightforward, it got me thinking about DNS resolution and how it is done inside the JDK.

For the impatient, here is a diagram:

If you want to know more, read on.

### Socket and InetAddress

As users of JDBC and other connection management libraries, we are abstracted from the low-level transport layer and communication protocol details as the libraries provide a convenient and rich API for us to work with. Most servers use TCP for communication and also have their own protocol to enable executing various commands sent from the client to the server. For example, these are the protocols for [Redis](https://redis.io/topics/protocol), [Postgres](https://www.postgresql.org/docs/9.3/protocol.html) and [MySQL](https://dev.mysql.com/doc/internals/en/client-server-protocol.html) which define the contents of the messages or commands, format and order.

[java.net](https://docs.oracle.com/javase/8/docs/api/java/net/package-frame.html) is where most of the network related classes are available in the JDK and when it comes to handling low-level communication between machines, it boils down to these two classes - [InetAddress](https://docs.oracle.com/javase/8/docs/api/java/net/InetAddress.html) and [Socket](https://docs.oracle.com/javase/8/docs/api/java/net/InetAddress.html). InetAddress is used to represent an IP address, be it IPv4 or IPv6. Socket defines an endpoint using an IP address and port number for communication between two machines. One of the constructors in the Socket class does exactly this — takes an InetAddress and port number to create a socket.

```java
public Socket(InetAddress address,
              int port)
       throws IOException
```

### Anatomy of host name resolution

To find the IP address of a host name, the code would be something like this:

```java
InetAddress address = InetAddress.getByName("www.somedomain.com"); 
System.out.println(address.getHostAddress());
```

`getByName` is where the resolution magic happens. I have learned to love reading OpenJDK source code and that is the right place to look to know more about this.

When this code is executed, there is a series of private method calls and this is what happens.

- The provided host value is checked if it is a `null` value. If it is, the [loopback](https://en.wikipedia.org/wiki/Loopback) address is returned — `127.0.0.1`
- If the input host is already an IP address, then there is no further lookup to do — input is returned
- Check if the hostname had been resolved earlier and stored in the cache. If it is available in the cache, the value is returned. The property [networkaddress.cache.ttl](https://docs.oracle.com/javase/7/docs/technotes/guides/net/properties.html) is used to control how long cached entries are valid. There is also a negative cache, which stores unsuccessful lookup so that the no time is spent on doing that again. `networkaddress.cache.negative.ttl` is used to set TTL for the negative cache.
- If the cache look up fails, all the registered [name services](https://docs.oracle.com/cd/E19455-01/806-1387/6jam6926f/index.html) are called one by one until the IP address is found.
- Unless custom name services are registered, there is a default name service provider registered which will be used for the look up.
- After these steps, from the name service it finally reaches a native method called [getaddrinfo](http://man7.org/linux/man-pages/man3/getaddrinfo.3.html) in Linux based systems. For Windows, you can imagine something similar is happening.

### getaddrinfo

[getaddrinfo](http://man7.org/linux/man-pages/man3/getaddrinfo.3.html) function is shipped as part of all operating systems conforming to the [POSIX](https://en.wikipedia.org/wiki/POSIX) standard. It is used for translating a host name into an IPv4 or IPv6 address. It is defined in the [specification](http://pubs.opengroup.org/onlinepubs/9699919799.2018edition/) as:

```
The getaddrinfo() function shall translate the name of a service location 
(for example, a host name) and/or a service name and shall return a set 
of socket addresses and associated information to be used in creating a 
socket with which to address the specified service.
```

In essence, it takes a host name and returns a list of IP addresses — yes, a list. I emphasize that here because, in the sample code earlier `java.net.InetAddress#getByName` returns a single address. When the native call to getaddrinfo is completed, the code in the JDK receives a list of addresses. All but one of the addresses are dropped and returned to the user’s code — as simple as `addresses[0]`.

Older versions of the JDK used a different function called [gethostbyname](http://man7.org/linux/man-pages/man3/gethostbyname.3.html) which is now obsolete. There are some excellent references about how `getaddrinfo` function works [here](https://jameshfisher.com/2018/02/03/what-does-getaddrinfo-do/), [here](https://github.com/angrave/SystemProgramming/wiki/Networking,-Part-2:-Using-getaddrinfo) and [here](https://beej.us/guide/bgnet/html/multi/syscalls.html#getaddrinfo) if you are interested in knowing more about the internals.

### Conclusion

Host name resolution is a complex topic and if you start following the clues, it takes you to interesting places. This was an opportunity for me to peek into OpenJDK code and some native functions. Hope you find this useful and don’t forget to set `networkaddress.cache.ttl` and `networkaddress.cache.negative.ttl` properties for your application.

> This post was originally [published in Medium](https://medium.com/@maheshsenni/host-name-resolution-in-java-80301fea465a) on Apr 16, 2019
