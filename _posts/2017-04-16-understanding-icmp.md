---
layout: post
title: "Understanding ICMP and why you shouldn't just block it outright"
date: 2017-04-16 00:00:00 +0000
author: Neil Alexander
redirect_from: "/articles/2017/4/16/understanding-icmp"
index: false
---

ICMP, or "Internet Control Message Protocol", is a protocol designed to help computers understand when things go wrong out on a network. It's a supporting protocol - that is, to say, that IP does not strictly require on ICMP to function, however typical networking devices such as routers and endpoints are expected to speak and understand ICMP. You might also know ICMP thanks to "ping", a utility designed to see if a remote computer on a network is alive and connected. 

You might also know about ICMP from some security guide that you read online which tells you, unwaveringly, to block ICMP traffic. "ICMP is a security risk," they chorus, "you must filter all ICMP packets from your network!"

Sadly, there are a staggering number of security professionals who actually know very little about the real working mechanisms of IP (and an even more staggering number that know nothing about Layer 2!) writing articles online and broadcasting this advice with a broken, or altogether missing understanding of what ICMP actually does. 

#### So what does ICMP do?

ICMP is a simple protocol. The packets are usually very small and typically contain a very limited amount of information. They also contain a type code that describes the purpose of the packet - in essence, the message being sent. There are a variety of type codes. ICMP is a bit of a diagnostic utility, and it is a bit of an error-reporting mechanism. It's used to tell a network device about a problem in the network when sending an IP packet.

You're probably familiar with "ICMP ECHO", or as it's known more affectionately, "ping". There are other codes that describe "ICMP DESTINATION UNREACHABLE" (for example, a host is offline, or there is no known route to it) and "ICMP TTL EXCEEDED" (the packet went over more hops than it was allowed to - actually, this is very useful for diagnosing routing loops!). There's a code to describe "ICMP BAD IP HEADERS" for a malformed IP packet, and some more to include information about "ICMP REDIRECTS" - this packet is no good here, send it there instead.

Network endpoints and routers should generate these ICMP messages as a result of one or more network conditions that may result in a packet not being delivered correctly to it's destination. That is, if you send a packet to an IP address for which no route exists, expect a router in the path to respond back to you with a "destination unreachable" ICMP message.

#### What are the consequences of blocking ICMP?

When you block ICMP, you are effectively filtering or dropping these warning packets from being delivered back to the sending endpoint. That IP packet that you sent off before? It never got there, but you'll never find out about it because the ICMP "destination unreachable" message was discarded before it reached you. Therefore your computer just assumes that nothing went wrong, and it will sit and wait, quite often for a long time, before giving up on expecting a response. This is known as a "timeout".

Had the "destination unreachable" packet made it back to you, you would know that there was a problem with the destination and your computer would give up instantaneously on that connection. You, or your application, would not be forced to wait (sometimes up to 60 seconds, or more!) for the connection to "timeout".

Another important ICMP packet that could be discarded by the filter is an "ICMP FRAGMENTATION REQUIRED" message. When that happens, things start to go really wrong.

#### Fragmentation? What's that?

So far we've seen the consequences of trying to talk to an unreachable host - whilst timeouts are inconvenient in that case, they are only masking another issue. However, blocking ICMP may even stop you from being able to communicate with reachable hosts too!

The key to this is that not all network links are created equal. To understand why, a bit of Layer 2 knowledge is required. In Ethernet land, a single "frame" of data (that is, a frame containing an IP packet) can only be so big. The default maximum frame size for an Ethernet network is 1500 bytes. Once you go above 1500 bytes, you have to create a new frame to send the next 1500 bytes. And so on, and so forth. To stream a large amount of information, you may send hundreds, thousands, or even hundreds of thousands of frames. 

Of course not all network links are based solely on Ethernet. Many broadband providers rely on a protocol called PPP to establish your connection across their network to the Internet, as this provides them with the extra authentication capability to identify you as a specific customer. PPP has additional headers, and when also wrapped in an Ethernet frame (this is known as PPPoE), means that there is less space available for your IP packet. Therefore 1500 bytes actually becomes 1492 bytes when you include the additional room needed to "fit" PPP onto the pipe.

This number - whether 1500, 1492 or any arbitrary number of bytes - is known as the "Maximum Transmission Unit", or MTU, and network devices must be aware of the MTU of a given link so that it does not produce frames too big for that link.

What happens when you send a frame that's 1500 bytes down a link that only supports frames of 1492 bytes? You guessed it - it won't fit. At this point, a router on the path has received a frame from a link where the MTU is 1500, and has tried to send it back out of another interface where the MTU is 1492.

Logically this is an impossible situation, so the router simply discards the frame and sends back a "fragmentation required" ICMP message back to the sender to say "This packet is too big for where you're trying to send it, so please make it smaller". The sending computer can then break down the packet into smaller chunks and resend them so that they'll now fit down the link. 

The sending computer will also "learn" from this ICMP message, temporarily remembering this condition for that given destination address, so the next packets that get sent will not exceed the given MTU size, avoiding the problem and allowing seamless communication back and forth. 

#### So if the sender never gets the "fragmentation required" packet...

... then the router on the path will discard the packet that's too big, it'll send an ICMP message back to you asking you to send smaller packets instead, that ICMP message will be blocked by your firewall and you will never get that memo, therefore your computer assumes that everything was fine.

In reality, it isn't fine because your packet was discarded by an upstream router so it simply never got there, and you never found out why because the warning ICMP packet was discarded before it got to you, therefore you sit and wait yet again until the "timeout" with absolutely no clue as to whether the data you sent actually got there or not. 

This is loosely known as "Path MTU discovery", and is a core feature of IP, to ensure that larger packets can be sent across different types of network link without too much trouble. By blocking ICMP, you are completely removing the computer's ability to learn about these conditions, creating unstable connectivity to that destination. 

####  Okay, so why do people claim it's such a security problem?

There are some legitimate reasons for believing that ICMP is a security issue, and there are plenty of outright myths.

One real problem is that "ICMP ECHO" packets (those used for "ping") can actually contain any arbitrary amount of information of pretty much any kind, which makes them useful for tunnelling packets inside "ICMP ECHOs" to avoid network boundary filtering using specialised software for this purpose. The reality of this is that most people are completely unaware that this is even a possibility, and some more intelligent firewalls may even be able to identify when this is happening.

Another is that "ICMP ECHO" actually reveals the existence of a device on the end of a given IP address very easily. The reality is that there are actually plenty of other ways to determine this, therefore this is a bit of a non-issue. Knowing that a machine exists doesn't really help you all that much - ICMP doesn't provide you with a "backdoor". You would still need some other route in by means of other open ports, and those open ports are just as likely to reveal the existence of the machine as "ping" is. 

Some are concerned that the "ICMP TTL EXCEEDED" packets may actually reveal the existence of routers on a path between two given hosts. This is actually the basis for how "traceroute" functions - send multiple packets with deliberately incremental TTL values, allow them to expire in transit and capture which routers report back with the "ICMP TTL EXCEEDED" warning.

Overly security-conscious individuals would prefer not to reveal the existence of things because it provides a kind of "security through obscurity", and therefore will just block all ICMP, not understanding that it has many functions outside of "ICMP ECHO". This is sadly a real world knowledge gap for a lot of IT professionals.

#### But I really really don't want people to be able to ping devices in my network.

Okay. In which case, what you need is to actually configure your firewall properly so that rather than blocking all ICMP traffic, you just simply block ICMP traffic with the specific type codes relative to "ping". For your information, those are "ICMP ECHO REPLY" (type code 0) and "ICMP ECHO REQUEST" (type code 8). 

That way, other diagnostic traffic, such as "ICMP DESTINATION UNREACHABLE" (type code 3) or "ICMP TTL EXCEEDED" (type code 11) will still be allowed through, and your computers will be able to learn about network problems properly. Hooray!

If you were particularly concerned about not revealing the existence of routers by means of "ICMP TTL EXCEEDED" (type code 11), then this specific message type could be filtered without too much ill-effect, at the expense of creating timeouts in genuine circumstances, i.e. when attempting to reconstruct fragmented packets or dropping packets where the TTL field has reached zero.

#### What else should I know about ICMP?

The problem is that even blocking "ping" can be bad in some scenarios - some software may use this mechanism to see if a host is available before trying to speak to it.

An extremely widely used example of this is Active Directory for domain-joined Windows clients, where ICMP is used to perform "slow link detection" before downloading and applying Group Policy Objects (GPOs). For more information on the consequences of ICMP on Active Directory, take a look at TechNet. Seeing problems with GPOs applying at logon? This might be related.

The other thing to be aware of is that ICMP is typically classed as "low priority" traffic by many routers and firewalls. That is, these devices should not prioritise ICMP traffic over typical IP traffic, therefore ICMP should really have little-to-no negative impact on the throughput of your network. However, if you were conscious about whether or not your network could be flooded with ICMP traffic, you can safely rate-limit ICMP, so long as you are not throttling it down so much that the ICMP packets end up being dropped regardless. 

#### In conclusion...

... there's a lot more to ICMP than initially meets the eye, and there are very real cases where ICMP is needed. Don't take the decision to block it outright lightly.
