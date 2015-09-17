---
tags: [swift, rack, ruby, server]
series: Web Development in Swift
series-index: 1
title: Demystifying the Server
abstract: Servers aren't a magical black box - they're just programs spitting out http responses. Let's have a look how a usual server infrastructure works.
---

# _Demystifying_ the _Server_

Servers aren't a magical black box - they're just programs spitting out http responses. Let's have a look how a usual server infrastructure works.

## The _Way_ of the _Request_

Here's what normally happens when you send a request to a server (no matter if it's via `curl` or `NSURLSession`):

- You send a `GET` request to `jensravens.de`
- A DNS server of your choice resolves the domain to an IP (192.30.252.153). Just think of it as a lookup in a phonebook that translates names into phone numbers.
- A request is sent to that IP (more on the request later) via tcp
- On the server (which is just a normal computer that is permanently connected to the internet) there is a process that listens on port 80 for tcp connections (80 is the default, but other ports would work as well). Usually that process is some dedicated web server like apache2 or nginx. It knows how to encrypt the connection via ssl, how http works and if the request is just for a plain file it's capable of delivering that file as a response.
- If you request a dynamic resource (like a php site, ruby scripts or whatever language you like) it calls out to another process to return a response.
- The external process does it's magic to create a http response.
- The response is sent back to the user via the proxy.

Of course it's also possible to skip the proxy and let your web application do the http handling directly. But normally you don't want to do that as it's pretty complex.

## The _Language_ of the _Web_

So how does that request look that is beeing sent? A very interesting part is the url of the request:

```
protocol://domain:port/path
```

The protocol normally is http or https for the web. Port is either 80 for normal http requests or 443 for encrypted https. The path is sent to the application to tell it which file it should create (it can of course contain parameters).

This is how a request looks that returns the front page of this blog (you can try it yourself by using `curl -v jensravens.de/`:

```
GET / HTTP/1.1
Host: jensravens.de
User-Agent: curl/7.43.0
Accept: */*
```

As you can see the request is just some plain text that is sent via tcp over the wire (in http2 it will be binary instead, but the format will not change). The first line states the http verb, the path (/ is default if nothing else is specified) and the http version (1.1 is in use for years now, but http2 is on the rise).

Next comes the host. So why should you as the client tell the server who he is? As you might remember from step 3 of the request the server is contacted via an IP, not by it's hostname. Therefore the server doesn't have to know it's own name. Also there can be multiple applications running on a single server.

The User-Agent tells the request who is asking for a file (server side detection of old Internet Explorer versions is mostly done through this header).

Accept tells the server what kind of format we're expecting as a return. Some valid examples would be `text/html` for html files, `application/json` which is mostly used in APIs or `image/jpeg` for images. `*/*` is a wildcard that tells the server: I don't care what you're sending, I can display it.

After two empty lines comes the request body (which can be gzip'd for better performance). But for a simple GET-Request there's no need for a body (you can place files here for file uploads or content for a post request).

After the server is done thinking what a good response is, it will send back a response (again clear text, I've stripped out some non important headers):

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 7897
Connection: Keep-Alive


<!DOCTYPE html>
<html lang="en">
...
```

Let's go through this line by line:

The first line confirms that the server likes to talk to us via the http 1.1 spec. Also the request was successfull (status `200 OK`). You can read all about status codes and their meaning at [HTTP Status Cats](https://www.flickr.com/photos/girliemac/sets/72157628409467125/) and at [Wikipedia](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes).

Next comes content type and length. After requesting `*/*` the server decided to send us a web page (`text/html`) with a content that has a length of 7897 bytes (the content follows at the end of the response).

Also it tells us what to do with the tcp connection that was opened for the request. Keep alive means that the client should reuse the same connection for further requests (like images). It can also be used to stream content to the client (more on that in a later post).

After two empty lines comes the content which in this case is just some plain text html (this is also called the body). The byte size of this chunk is what was specified in the content length header.

## _Look Mommy_, I made a _Webserver_!

So you might think now: Hey, that's just some plain text sent over a socket, I can do that myself! And you're absolutely right. Writing a simple http server is pretty easy and there are some libraries that help you like [CocoaAsyncSocket](https://github.com/robbiehanson/CocoaAsyncSocket) and [CCLHTTPServer](https://github.com/cocodelabs/CCLHTTPServer). But they all require to tighly couple your application code to network code (which makes it close to impossible to replace the network stack). Other languages had similar problems so the ruby community made [rack](http://rack.github.io/), a standard interface for ruby web applications that splits networking and request handling into manageable chunks. In the next part we'll explore which options there are for Swift and Objective C.
