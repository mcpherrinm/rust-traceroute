Writing Traceroute (In Rust!)
=============================

I thought writing my own Traceroute might be a fun little project to do one
afternoon.  While I understand the idea behind it, I have never tried to
implement it and thought it to be a worthwhile exercise.  Since I think I know
how traceroute works, I would expect this to be mostly straightforward!

There are a number of enhancements and design decisions that go along with a
program like traceroute, so this should be a reasonable exercise in writing
real code in Rust.


What is Traceroute?
-------------------

Traceroute is a traditional netowrking tool that tells you, roughly, what the
routers along the path a packet is taking to some destination.  How it works
is pretty simple, and based on the fact that IP packets have a header field
called "Time To Live".  Each time a router passes a packet, it decrements this
by one, until it is either delivered or hits zero, where it is dropped.  Most
of the time, the router dropping the packet will be nice enough to send you a
little note telling you it did so.  This prevents a packet from getting stuck
in the network and looping around forever and ever, which is pretty important
since the internet is a pretty complicated place and routing loops do happen!

We can use this behaviour to figure out what routers are along the path by
sending a bunch of packets with incrementing TTLs.  The first one we send
has TTL=1, so the first router drops it and lets us know.  The second packet
has TTL=2, and so forth, until we get to the end of our path.

We can time how long it takes the packets to be returned too, which gives us
some idea how much latency is on each link of the path.  This can be deceiving
though: Routers forward packets through dedicated hardware, and the ICMP
unreachable messages come from the controlling CPU, which may take a little
longer, especially if it's busy.


What networking APIs do I need?
-------------------------------

I'm going to be writing this in Linux.  There are two important tasks we need
the networking stack to do for us.  First, we need to be able to send packets
with a TTL we can choose.  Then, our program needs to get the ICMP messages
sent back.  We'll probably also want some other features like DNS resolution
for a more complete program, but I'll leave that for later.

Rust's networking library seems to support sending UDP packets with a chosen
TTL, so that seems like a reasonable place to start sending.  To recieve the
ICMP replies, I'll need a raw socket, which Rust doesn't have standard library
support for, so I'll write a little wrapper library around the Linux Socket
functions.


Sending some packets
--------------------

Let's start by writing a small program to send UDP packets with a desired TTL.
A quick search of the Rust documentation for ttl reveals the [set_ttl] method
on a [UdpSocket].  There's a sample program in its documentation, so I'll
start by modifying it.

[sample-send-udp.rs](sample-send-udp.rs)

You can use tcpdump to see that this program sends packets and gets ICMP time
exceeded replies from the routers along the path.  Trying to use tcpdump to
view the results isn't exactly convenient, so next up is to listen for the
replies.


Receiving ICMP packets
----------------------

Receiving ICMP replies is a little tricky.  Unlike UDP or TCP, the Operating
System's abstractions of communication over a pipe breaks down.  We need to
grab all the incomping ICMP messages and figure out which are applicable for
us.  Because we're listening to all of them, the program has to have root
privileges. Traceroute and ping are often setuid for this reason.

The Rust standard library doesn't have a safe interface for raw sockets.  We'll
have to use the libc wrappers.  I can never remember the API for sockets on
Unix, so it's off to `man 7 raw` for my quick refresher.  Unfortunately,
liblibc doesn't seem to have a binding for SOCK_RAW and IPPROTO_ICMP, which we
need to pass to our call to `socket`.  I'll just hard-code them for now, based
on values grepped out of /usr/include

With the socket opened, we need to get packets out of it.  The Rust UdpSocket
in libnative has a recvfrom method that does almost the same thing.  Our code
will look pretty similar.  Because recvfrom takes pointers and lengths, we'll
write a safe wrapper to keep out misuses of buffers, a common source of bugs in
C programs.

We'll test out these APIs by writing a small program that lists the source and
dumps the contents of all incoming ICMP packets.  Run this program and then
run the previous sample to see all the replies come in.  While we can't easily
understand the responses yet, you should be able to pick out the involved IP
addresses from the bytes.

[sample-receive-icmp.rs](sample-receive-icmp.rs)



Understanding ICMP responses
----------------------------

We're getting some bytes in a buffer in our ICMP receive sample, so let's try
to make sure they look vaguely sane.  We can copy and paste a few samples into
a text file and get the basic data out of them.

Skimming [RFC 792](http://tools.ietf.org/html/rfc792) for their pretty ASCII
pictures of the ICMP header formats, it looks like it the format is very
simple, and based around easy power-of-two sized fields.  This sort of stuff is
really conveniently parsed with Rust's Reader trait.  Since we've got the data
to parse in memory, we'll wrap it in a BufReader and start sucking out our
data!

First, we can parse out the IP header.  Technically, if the header includes
options, it isn't fixed length, but since this is just a little sample program
we ignore this case and just check the header length is 5 words.  We'll make
sure the protocol is ICMP, and grab the source IP address too.

Then we'll grab the type out of the ICMP, and the TTL from the original IP
header.  For now we won't bother with nice output, and will just print them
in the order they arrive, which should be good enough.

[sample-parse-icmp.rs](sample-parse-icmp.rs)


Librarification of packet parsing
---------------------------------

Now we've got all the ingredients, we can put together the basic traceroute
program.  We'll start with taking the minimal ICMP parser sample and making it
into something usable as a library.  It'll consist of functions that pull
each header out of a stream you pass in.

The headers in the response payload may be truncated, but we don't need really
need to handle that: all the responses we get back should at least have the IP
and UDP headers intact.  But we should do proper error checking.

[packet.rs](packet.rs)
