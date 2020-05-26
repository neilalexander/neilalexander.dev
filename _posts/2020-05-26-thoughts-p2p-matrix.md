---
layout: post
title: "Thoughts on Peer-to-Peer Matrix"
date: 2017-05-26 00:00:00 +0000
author: Neil Alexander
index: false
sitemap: false
---

Since starting at New Vector in December 2019, I have been working mostly on
Dendrite, a Matrix homeserver written in Go. Part of this is to hopefully produce
a Matrix homeserver implementation that can stand as a fully feature-complete
alternative to Synapse. However, we've also been using Dendrite as a testbed for
an entirely new model of Matrix federation which is fully peer-to-peer.

Currently, users register their accounts on a homeserver which may be owned and
operated by someone else. The goal is to shift this model around so that instead
users will interact with the Matrix network by running their own single-user
homeservers right on the devices that they are using.

This has a number of potential benefits. It makes it possible to use Matrix in
situations where normal Internet connectivity may not be available or reliable.
It makes it possible for users to own their own data, rather than entrusting it
to a homeserver. It makes it simpler for people to join Matrix without having to
worry about homeservers, data retention etc.

As an entertaining example of this, we compiled Dendrite for WebAssembly and
built it into a customised version of Riot Web. Using the JavaScript libp2p 
libraries, and with the help of a rendezvous server, you can open up p2p.riot.im
in a web browser and use Riot, except instead of logging into a remote homeserver,
you are actually logging into a homeserver that is running right there in your web
browser, locally. You can create rooms and talk to other users of the P2P demo
pretty much transparently.

This is by no means a final representation of what peer-to-peer Matrix will look
like, but rather it is one of two initial experiments into using libp2p as a
carrier for Matrix federation traffic. (The other was discussed at FOSDEM 2020,
using the libp2p-go bindings and multicast DNS for discovery on the local network.)

On the one hand, it was relatively painless building the libp2p demos as many of
the needed components for HTTP-over-libp2p were already available. However, there
are still a number of issues with this approach. Inside the browser sandbox, we
are limited to only being able to communicate using WebSockets, WebRTC and similar,
hence why all traffic in the P2P demo is in reality proxied through the rendezvous
server in a star topology. Peer-to-peer in spirit, perhaps, but not truly.

The FOSDEM P2P demo, on the other hand, took a different approach. It still used
libp2p, but instead of using the JavaScript bindings inside the browser, it instead
used the Go bindings and ran outside of the browser as a standalone executable.
There is no rendezvous server either - instead, multicast DNS is used to automatically
find and connect to other nodes on the same network segment. This means that you can
throw a bunch of users onto the same Wi-Fi network and they will be able to talk to
each other directly, without relying on any third-party software. The experience is
less seamless due to the fact that the homeserver has to be started separately at
this stage, and it is rather limiting in that you can only talk to people nearby,
but it gets much closer to the real peer-to-peer vision of not relying on a central
server.

The problem with both of these approaches is that neither of them truly have the
ability to handle overlay routing - this is a rather big gap in libp2p at present.
libp2p will work fine in scenarios where you can guarantee that your participating
nodes will be able to reach each other directly using the underlying network, but
things quickly fall apart when you cannot guarantee this being the case (e.g. in
ad-hoc settings). Since federation within a Matrix room is effectively full-mesh,
you need to be able to exchange data with all servers that are participating in a
given room directly. 

So the next likely iteration is to consider whether or not Yggdrasil will help us.
Yggdrasil is a decentralised routing scheme which aims to guarantee end-to-end
network connectivity between all network nodes by routing traffic on behalf of users
who cannot communicate directly. It uses a global spanning tree as a locator 
mechanism and a distributed hash table (DHT) to facilitate two nodes finding each
other. Yggdrasil is much more bare-bones than libp2p in many ways - there's no
pre-packaged HTTP handlers, for example. Instead you just get a socket-like structure
and Yggdrasil will try to get packets from A to B.

However, we gain full overlay routing and the ability to connect nodes in pretty
much any network topology (including over the internet, over local area networks and
even directly using wireless) and, moreover, we should be able to do so at scale.
Some work will need to be done to create the HTTP boilerplate however, although I
have done some preliminary work on HTTP/2 multiplexing over Yggdrasil which may fit
this use-case nicely.

It's worth noting though that Yggdrasil is not perfect either. I have been one of two
maintainers on the Yggdrasil Network project for over two years now and, although
many of the design elements are sound, there are still weaknesses in the protocol
design that need to be addressed in order for it to be truly resilient. Currently,
to avoid needing full routing tables, Yggdrasil assigns "coordinates" to a node
relative to a root node on the network, which is chosen based on the attributes of the
cryptographic keys. Routing is then performed by sending traffic to a peer that takes
your traffic closer to the destination coordinates. However, it's possible for a
root node to go offline or to be replaced at pretty much any time, causing all other
nodes on the network to reassign their coordinates and perhaps being briefly
unreachable to each other while this happens. It's also entirely possible that a bad
actor on the network can selectively and maliciously drop traffic whilst transiting 
for other nodes, and we don't currently have a means for avoiding this.

One area of exploration which may help to solve these problems is to examine whether
source routing between two communicating nodes can increase resilience. Source routing
is where the path that the traffic will follow is calculated by the sender up front,
rather than relying on each node on the path to make a routing decision. This means
that the sender can specifically choose a different route if someone on the path 
decides to start dropping traffic. Source routing would also be far less reliant on
the node's coordinates as the path would be calculated based on the switch ports on
the path, therefore network topology changes further up the spanning tree (such as
those that would result in changes in coordinates) would not result in a loss of
connectivity since the path would otherwise not change. There is a balance to be
struck here though in that source routing searches may end up being more expensive
up front and it may be a lot harder to respond to network conditions on the path,
e.g. by trying to route around network congestion. 

In addition to connectivity, there are other Matrix-centric problems to be solved.
In a world where each device is potentially running its own homeserver, we will need
a new approach for handling user identities. A draft proposal exists today in the 
form of MSC1228 which aims to replace traditional Matrix user accounts with portable
identities which are cryptographically generated. Not covered yet by the MSC is how
we will maintain a synchronised state between multiple devices owned by the same user,
since the identity will effectively need to be "portable" before that can be achieved.

It's also possible that a user's device will be unreachable at pretty much any time,
either because it has no network connectivity, it has ran out of power or has even
just been switched off by the owner. Therefore, we will also be exploring whether 
other more-static homeservers can be used as a type of mailbox for storing messages
on behalf of a user while their own devices are offline. 

P2P Matrix is still very much on the drawing board at this stage - this is just a
taste of some of the issues still to be worked out and I expect that we will iterate
on many designs before we arrive at a final destination. However, this is a project
that has a lot of interest and an even higher level of potential. I look forward to
being able to discuss more on our findings as and when we explore different options!