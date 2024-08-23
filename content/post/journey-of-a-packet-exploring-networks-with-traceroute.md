---
title: "The journey of an internet packet: Exploring networks with traceroute"
date: 2024-08-23T00:00:00-06:00
summary: "This article explains the mechanics behind traceroute, from IP headers and TTL fields to ICMP messages. Learn why traceroute is a crucial tool for network diagnostics, how it differs from ping, and how to interpret its output. Ideal for network administrators, IT professionals, and curious users looking to understand network topology and troubleshoot connectivity issues. Gain insights into internet infrastructure and enhance your ability to diagnose network problems efficiently."
author: "Sebastian Marines"
categories: ["networking"]
tags: ["networking"]
---

We often take for granted that when we try to connect to a server, the connection will work. But that is not always the case. Sometimes there is something broken with the network that prevents us from reaching the destination. But how can we know where the problem is?

I often found myself using `ping` to test connectivity between two machines, and while `ping` is a great tool to test if a machine is reachable, it doesn't give us much information about what can be wrong.

The internet is a complex network of routers, switches, and computers, and when we try to connect to a server, our packets go through many routers before reaching the destination. If one of these routers is misconfigured or down, the packet can be dropped, and we can't reach the destination.

In this post, we will see how `traceroute` works, and how it can help us diagnose network problems.

![Ping between two machines](/traceroute/simple-ping.svg)

## The life of a packet

When you connect to a server, either a website or any other service, your computer checks if the destination IP address is in the same network. If it is, it sends the packet directly to the destination. If it is not, it sends the packet to the default gateway, which is your ISP's router that connects your network to the internet.

The router then checks if the destination IP address is in its routing table. If it is, it sends the packet directly to the destination. If it is not, it sends the packet to the next router in the path. This process repeats until the packet reaches the destination.

In the next image you can see this process. First, the computer with an IP address of `10.0.0.1` pings the computer with an IP address of `10.0.0.2` sending an ICMP packet. Since they are not in the same network, the packet is sent to the router with an IP address of `10.1.0.5`. This router then forwards the packet to the router with an IP address of `10.1.0.6`, which forwards the packet to the computer with an IP address of `10.1.0.7`. The last router finally sends the packet to the destination computer, and the packet reaches its destination.

![Ping between two machines with multiple routers](/traceroute/ping-multiple-routers.svg)

If we ping the destination computer we can see that it responds correctly.

```bash
$ ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: icmp_seq=0 ttl=58 time=5.368 ms
64 bytes from 10.0.0.2: icmp_seq=1 ttl=58 time=4.325 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=58 time=4.225 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=58 time=4.479 ms
```

## What happens when there is a problem?

But now, what happens if there is a problem in the path? For example, what happens if one of the routers is misconfigured and drops the packet? Or what happens if the destination computer is down? How can we know where the packet is being dropped?

Take a look at the following example, where the router `10.1.0.6` can't communicate with the router `10.1.0.7`.

![Ping between two machines with a broken path](/traceroute/ping-broken-path.svg)

If we ping the destination computer we can see that we can't reach the target, but we don't know where the problem is.

```bash
$ ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
^C

--- 10.0.0.2 ping statistics ---
4 packets transmitted, 0 received, +4 errors, 100% packet loss, time 3031ms

```

This is where `traceroute` comes in handy. `traceroute` is a network diagnostic tool that shows the path that a packet takes from the source to the destination. It allows us to see every router in the path and the response time of each one.

```bash
$ traceroute 10.0.0.2
traceroute to 10.0.0.2 (10.0.0.2), 30 hops max, 60 byte packets
 1  10.1.0.5  1.123 ms  0.912 ms  1.145 ms
 2  10.1.0.6  2.145 ms  2.023 ms  2.311 ms
 3  * * *
 4  * * *
 5  * * * 
```

In this example we can see that routers `10.1.0.5` and `10.1.0.6` are working correctly, but there is a problem with the next router in the path. You can see that we first reach the router `10.1.0.5` and the response time is around 1ms. Then we reach the router `10.1.0.6`, with a response time of around 2ms. But then, the path is broken, and we can't reach the destination computer.

## How does `traceroute` work?

To understand how `traceroute` works, we first need to understand the fields in an IP header.

![IPv4 header](/traceroute/ipv4-header.svg)

There are several fields in the IP header, but we will focus on 3 of them:

- **Source IP address**: The IP address of the computer that sends the packet.
- **Destination IP address**: The IP address of the computer that receives the packet.
- **Time to live (TTL)**: The maximum number of routers that the packet can pass through.

The source and destination IP addresses are self-explanatory, but the TTL field is an interesting one. This field is used to prevent packets from looping indefinitely in a network.

When a packet is sent, the TTL field is set to a value of 64 (at least in Linux and macOS). When the packet reaches a router, the router decrements the TTL field by 1. If the TTL field reaches 0, the router drops the packet and sends an *Time Exceeded Message* back to the source computer. But if the TTL field is greater than 0, the router forwards the packet to the next router in the path.

This is how `traceroute` is able to show the path that a packet takes from the source to the destination. It sends an ICMP packet with a TTL of 1, then a packet with a TTL of 2, then a packet with a TTL of 3, and so on. This way, `traceroute` can see the response of each router in the path.

## What is ICMP?

Up to this point, we have mentioned ICMP a couple of times, but what is ICMP?

ICMP stands for *Internet Control Message Protocol*. It is a protocol used by network devices to diagnose network communication issues[^1].

[^1]: [What is the Internet Control Message Protocol (ICMP)?](https://www.cloudflare.com/learning/ddos/glossary/internet-control-message-protocol-icmp/)

This protocol is used by commands like `ping` and `traceroute` to test network connectivity.

The ICMP standard[^2] defines several message types. The most common ones are:

[^2]: [Internet Control Message Protocol](https://datatracker.ietf.org/doc/html/rfc792)

- **Echo Request**: Used by the `ping` command to test if a computer is reachable.
- **Echo Reply**: Sent by a computer in response to an *Echo Request*.
- **Time Exceeded Message**: Sent by a router when the TTL field of a packet reaches 0.
- **Destination Unreachable Message**: Sent by a router when the destination computer is unreachable.
- **Redirect Message**: Sent by a router to inform a computer that there is a better route to a destination.
- **Parameter Problem Message**: Sent by a router when a packet has an incorrect header.
- **Source Quench Message**: Sent by a router to inform a computer that it is sending too many packets.


## Visualizing `traceroute`

Let's run `traceroute` to the destination computer `10.0.0.2` and see what happens in each step.

```bash
$ traceroute 10.0.0.2
traceroute to 10.0.0.2 (10.0.0.2), 64 hops max, 40 byte packets
...
```

First, `traceroute` sends an ICMP packet with a TTL of 1. The packet reaches the first router in the path, which decrements the TTL field by 1. Since the TTL field is now 0, the router drops the packet and sends an *Time Exceeded Message* back to the source computer.

![Ping with a TTL of 1](/traceroute/ping-ttl-1.svg)

```bash
$ traceroute 10.0.0.2
traceroute to 10.0.0.2 (10.0.0.2), 64 hops max, 40 byte packets
 1 10.1.0.5 (10.1.0.5)  1.123 ms  0.912 ms  1.145 ms
...
```

> You can see that we have 3 response times for the first router. This is because `traceroute` sends 3 packets to each router.

Next, `traceroute` sends an ICMP packet with a TTL of 2. The packet reaches the first router in the path, which decrements the TTL field by 1. The TTL field is now 1, so the router forwards the packet to the next router in the path. 

The packet reaches the second router in the path, which decrements the TTL field by 1. Since the TTL field is now 0, the router drops the packet and sends an *Time Exceeded Message* back to the source computer.

![Ping with a TTL of 2](/traceroute/ping-ttl-2.svg)

```bash
$ traceroute 10.0.0.2
traceroute to 10.0.0.2 (10.0.0.2), 64 hops max, 40 byte packets
 1 10.1.0.5 (10.1.0.5)  1.123 ms  0.912 ms  1.145 ms
 2 10.1.0.6 (10.1.0.6)  2.145 ms  2.023 ms  2.311 ms
...
```

Now, `traceroute` sends an ICMP packet with a TTL of 3. The packet reaches the first router in the path, which decrements the TTL field by 1. The TTL field is now 2, so the router forwards the packet to the next router in the path.

When the packet reaches the last router in the path, the router decrements the TTL field by 1. Since the TTL field is now 0, the router drops the packet and sends an *Time Exceeded Message* back to the source computer.

![Ping with a TTL of 3](/traceroute/ping-ttl-3.svg)

```bash
$ traceroute 10.0.0.2
traceroute to 10.0.0.2 (10.0.0.2), 64 hops max, 40 byte packets
 1 10.1.0.5 (10.1.0.5)  1.123 ms  0.912 ms  1.145 ms
 2 10.1.0.6 (10.1.0.6)  2.145 ms  2.023 ms  2.311 ms
 3 10.1.0.7 (10.1.0.7)  3.145 ms  3.023 ms  3.311 ms
...
```

Finally, `traceroute` sends an ICMP packet with a TTL of 4. The packet reaches the first router in the path, which decrements the TTL field by 1. Then each router in the path decrements the TTL field by 1 until the packet reaches the destination computer.

The destination computer receives the packet and sends an *ICMP Echo Reply* back to the source computer.

![Ping with a TTL of 4](/traceroute/ping-ttl-4.svg)

```bash
$ traceroute 10.0.0.2
traceroute to 10.0.0.2 (10.0.0.2), 64 hops max, 40 byte packets
 1 10.1.0.5 (10.1.0.5)  1.123 ms  0.912 ms  1.145 ms
 2 10.1.0.6 (10.1.0.6)  2.145 ms  2.023 ms  2.311 ms
 3 10.1.0.7 (10.1.0.7)  3.145 ms  3.023 ms  3.311 ms
 4 10.0.0.2 (10.0.0.2)  4.145 ms  4.023 ms  4.311 ms
```

## Conclusion

Traceroute is a powerful diagnostic tool that helps us understand the path our data takes through the internet. It provides valuable insights into network topology and potential issues along the way, and allows us to:

- Identify where packets are being dropped or delayed
- Discover unexpected routing paths
- Detect misconfigured or malfunctioning routers
- Estimate network latency between hops

While ping can tell us if a destination is reachable, traceroute shows us the entire journey, making it an essential tool in any network administrator's or curious user's toolkit.

As we've seen, the internet is a complex web of interconnected devices, and traceroute helps simplify this complexity. Next time you encounter a network issue, remember to use traceroute, it might help you find the problem and save valuable troubleshooting time.
