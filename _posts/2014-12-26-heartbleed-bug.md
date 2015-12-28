---
layout: post
title:  "The heartbleed bug"
date:   2014-12-26 02:10:23
categories: security c bug
---

![Heartbleed]({{ site.url }}/assets/heartbleed.jpg)

I am sure you have all heard about the "Heartbleed" bug affecting OpenSSL recently,
and many people around me ask me about this bug, so I decided to make an article
 so that everyone can understand how it works. It is actually quite simple to
understand, so even if you don't read the technical details you should be able
to understand.

### Non-technical explanation

First, OpenSSL is an open-source software which aims to implement the TLS
protocol (formerly called the SSL protocol). The TLS protocol was created to
provide an efficient way to establish a secure connection between two points so
that no one can normally know what those two point are "talking" about together.
If you go to your bank website, the URL should be something like "https://"
(if not, consider changing bank :D) that means you are communicating safely with
your bank over the TLS protocol.

The problem is that every time you make a request to the server (when you go to
another page for example) you have to re-establish a secure connection, and it
takes time. Developers then decided to create a "keep-alive" system, allowing
the user to regularly specify to the server that it is still here, so the server
won't close the secure connection, and it won't have to be re-created after.
This "keep-alive" system has been called "Heartbeat".

Let's see how heartbeat works:

 1. The client sends a message (a packet) to the server. This packet stands for
"I'm still alive, do not close the connection". This message contains two
important info: a randomly generated string and its size.
 2. The server receives this packet and stores it in memory for a few time
(a few ms) so he can finish what he was doing. Once finished, he looks in his
memory to find the sent message, and reads it to send it back to the client.
He knows when he finished reading, because the client sent him the size of the
message.
 3. The client receives the message he sent to the server a few ms ago,
everything is working fine, the connection is kept alive.

Do you see the flaw here? There's no server-side control to check if the sent
message is actually the same size as the sent size. So if the user specify a
greater size than the message's size, the server will work the same way, and it
will return some of its memory's content located after the message!
And it is a huge problem, because in this memory there can be passwords,
private data, files etc.

Here's a little comic from xkcd.

![Heartbleed explanation]({{ site.url }}/assets/heartbleed_explanation.png)

### On the technical side

Here's what the Heartbleed bug looks in the code[^2]

{% highlight c linenos %}
unsigned char   *p = &s->s3->rrec.data[0], *pl;
unsigned short  hbtype;
unsigned int    payload;

hbtype = *p++;
n2s(p, payload);
pl = p;
{% endhighlight %}

Here, `p` points to the received data, `hbtype` represents the type of the message
(either request or response), `payload` is the size of the payload
(the message contained in the received packet).

The line 5 reads the first byte of the packet (the type of the message), makes `p` point
on the next byte, and sets `hbtype` to the type of message.

The line 6 reads the next two bytes[^1] (which represent the length of the payload), copies
them to the `payload` variable and makes the pointer point two bytes further.
`pl` is a pointer to the payload.

Here is how the server replies:

{% highlight c linenos %}
unsigned char   *buffer, *bp;
unsigned int    write_length = 1 +
                               2 +
                               payload +
                               padding;

buffer = OPENSSL_malloc(write_length);
bp = buffer;
*bp++ = TLS1_HB_RESPONSE;
s2n(payload, bp);
memcpy(bp, pl, payload);
{% endhighlight %}

Here, the length of the response is computed like :

 * 1 byte to store the type of the message (request or response)
 * 2 bytes to store the length of the payload
 * `payload` bytes to store the payload
 * `padding` bytes so the message has a fixed length

Line 7 allocates the memory needed for the response, line 8 makes `bp` point to
this memory area.

Line 9 writes the type to the response, and makes `bp` point to the next byte.

Line 10 writes the size of the payload (two bytes) to the response, and shift
the pointer by two bytes, so it now points on the message.

Line 11 copies the sent message to the response on `payload` bytes. Here is the flaw.
The size was never checked on the server-side, so we rely on the size sent by
the client to read from the server's memory.

Therefore, it is really easy to get interesting data from the server, you just
have to specify the maximum length for the message (0xFFFF), and to write an
empty or a really short message, and the response will contain the next 64kB
after the message in the server's memory (possibly credentials, files, ...)

### Conclusion

Here are some lessons we can learn from the Heartbleed bug:

 1. be really careful when treating sensible data with low-level languages where
you have to manually handle the memory.
 2. use proper name for variables, do not hesitate to write lengthy comments,
this way other developers can quickly understand what the code is doing, and
reveal issues before they are being exploited.
 3. do not skip the testing phase. I am sorry to say that, but This bug could
have been easily discovered by sending random values to the server and by
comparing to the response to the request.


#### Notes

[^1]: here you can see that, fortunately, the size is encoded on two bytes, so the size can be up to <span>$$ 2^{16} = 65536$$</span> bytes, ~64kB). Thus the leak cannot be bigger than 64kB. That is why you should carefully choose your data types before writing code. Imagine how worse the bug would have been is the size was encoded on 4 bytes (~4MB)
[^2]: the code can be found on [Github][github]

[github]: https://github.com/openssl/openssl/blob/731f431497f463f3a2a97236fe0187b11c44aead/ssl/d1_both.c
