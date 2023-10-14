This chapter deals with different kinds of issues in distributed system 

Single node is *deterministic* where same operation always produces the same result

In distributed systems, there may be some part of the system that are broken which is known as *partial failure*. Partial failure are *nondeterministic*

Because distributed system are connected by network, you may not even know whether something succeeded or not, as the time it takes for a message to travel across a network is also nondeterministic

Large scale computing system has two kinds
- High performance computing
- Cloud computing which associated with multi-tenant datacenters
This book focus on systems for implementing internet services which has following entities:
- Application are online which means stopping the cluster is not acceptable
- Nodes in cloud services are built from commodity hardware which means higher failure rates.
- It is reasonable to assume that something is always broken.
## Unreliable Networks
Distributed system is build on top of network (shared nothing architecture) 
And network is inherently unstable. 

The internet and most internal networks in datacenters are *asynchronous packet networks* which means when one node sends a message, network gives no guarantees to when it arrive or it arrive at all. 

If you send a request and expect a response, many things could go wrong:
1. Your request may have been lost 
2. Your request may be waiting in a queue
3. Remote node may have failed
4. Remote node may temporarily stopped responding (long garbage collection) 
5. Remote node may processed the request but response get lost 
6. Response may get delayed (network on receiver's machine is overloaded)

![[different_error_in_network.png]]

#### Network Faults in Practice
>during a software upgrade for a switch could trigger a network topology reconfiguration, during which network packets could be delayed for more than a minute [17]. Sharks might bite undersea cables and damage them [18]. Other surprising faults include a network interface that sometimes drops all inbound packets but sends outbound packets successfully [19]: just because a network link works in one direction doesn’t guarantee it’s also working in the opposite direction.

[17] Mark Imbriaco: “[Downtime Last Saturday](https://github.blog/2012-12-26-downtime-last-saturday/),” github.com, December 26, 2012.
[18] Will Oremus: “[The Global Internet Is Being Attacked by Sharks, Google Confirms](https://slate.com/technology/2014/08/shark-attacks-threaten-google-s-undersea-internet-cables-video.html),” slate.com, August 15, 2014.

<iframe width="560" height="315" src="https://www.youtube.com/embed/4WlOnlRncK0?si=BtEdwepsqw2J1uK_" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

>If the error handling of network faults is not defined and tested, arbitrarily bad things could happen: for example, the cluster could become deadlocked and permanently unable to serve requests, even when the network recovers [20], or it could even delete all of your data [21]. If software is put in an unanticipated situation, it may do arbi‐ trary unexpected things.

#### Detecting Faults
- Load balancer needs to stop sending requests to a dead node
- In distributed database with single leader replication, if leader fails, one of follower needs to be promoted to the new leader

In some circumstances, we might get feedback to tell us something is not working
- OS will refuse TCP connections by sending a `RST` or `FIN` packet in reply
- If a node process crashed (or killed by admin), node's OS can run a script to notify other nodes about the crash so another node can take over quickly without having to wait for timeout to expire 
- If we have access to interface of the network switches in our datacenter, we can query them to detect link failures at hardware level (this option is ruled out if we are connecting via the internet) 
- If router is sure the IP address is unreachable, it may reply with ICMP Destination Unreachable packet

>Conversely, if something has gone wrong, you may get an error response at some level of the stack, but in general you have to assume that you will get no response at all. You can retry a few times (TCP retries transparently, but you may also retry at the application level), wait for a timeout to elapse, and eventually declare the node dead if you don’t hear back within the timeout.

#### Timeouts and Unbounded Delays 
If timeout is only sure way of detecting a fault, then how long should the timeout be? 

Long timeout means long wait, short timeout may risk of incorrectly declaring a node is dead 

>Prematurely declaring a node dead is problematic: if the node is actually alive and in the middle of performing some action (for example, sending an email), and another node takes over, the action may end up being performed twice.

Imagine a fictitious system with a network that guaranteed a maximum delay for packets—every packet is either delivered within some time d, or it is lost, but delivery never takes longer than d. Furthermore, assume that you can guarantee that a nonfailed node always handles a request within some time r. In this case, you could guarantee that every successful request receives a response within time 2d + r—and if you don’t receive a response within that time, you know that either the network or the remote node is not working. If this was true, 2d + r would be a reasonable timeout to use.

Unfortunately, most systems we work with have neither of those guarantees: asynchronous networks have unbounded delays

##### Network congestion and queueing 
Same as traffic congestion, packet delays varies with network due to queueing:
- If several different nodes send packets simultaneously to same destination, the network switch must queue them up. If there is so much incoming data that switch queue fills up, the packet is dropped 

![[packet_drop.png]]

- When a packet reaches the destination machine, if all CPU cores are currently busy, the incoming request from the network is queued by the operating system until the application is ready to handle it.
- In virtualized environments, a running operating system is often paused for tens of milliseconds while another virtual machine uses a CPU core. During this time, the VM cannot consume any data from the network, so the incoming data is queued (buffered) by the virtual machine monitor [26], further increasing the variability of network delays.
- TCP performs *flow control* (also known as *congestion avoidance* or *backpressure*), in which a node limits its own rate of sending in order to avoid overloading a network link or the receiving node [27].

#### TCP vs UDP
>Some latency-sensitive applications, such as videoconferencing and Voice over IP (VoIP), use UDP rather than TCP. It’s a trade-off between reliability and variability of delays: as UDP does not perform flow control and does not retransmit lost packets, it avoids some of the reasons for variable network delays (although it is still susceptible to switch queues and scheduling delays).
  UDP is a good choice in situations where delayed data is worthless. For example, in a VoIP phone call, there probably isn’t enough time to retransmit a lost packet before its data is due to be played over the loudspeakers. In this case, there’s no point in retransmitting the packet—the application must instead fill the missing packet’s time slot with silence (causing a brief interruption in the sound) and move on in the stream. The retry happens at the human layer instead. (“Could you repeat that please? The sound just cut out for a moment.”)

##### Choice timeout experimentally
In public clouds and multi-tenant datacenters, resources are shared among many customers: the network links and switches, and even each machine’s network interface and CPUs (when running on virtual machines), are shared. Batch workloads such as MapReduce (see Chapter 10) can easily saturate network links. As you have no control over or insight into other customers’ usage of the shared resources, network delays can be highly variable if someone near you (a *noisy neighbor*) is using a lot of resources [28, 29].

In such environment, we need to choose timeouts experimentally: 
measure the distribution of network round-trip times over an extended period and over many machines. Then determine the expected variability of delays 

### Synchronous vs Asynchronous Networks
Why can’t we solve this at the hardware level and make the network reliable so that the software doesn’t need to worry about it?

>To answer this question, it’s interesting to compare datacenter networks to the traditional fixed-line telephone network (non-cellular, non-VoIP), which is extremely reliable: delayed audio frames and dropped calls are very rare. A phone call requires a constantly low end-to-end latency and enough bandwidth to transfer the audio sam‐ ples of your voice. Wouldn’t it be nice to have similar reliability and predictability in computer networks?

When you make a call over telephone network, it establishes a *circuit*: fixed, guaranteed amount of bandwidth is allocated for the call, along with the entire route between the two callers. This circuit remains in place until the call ends. 

This kind of network is *synchronous* 

#### Can we not simply make network delays predictable? 
Circuit in a telephone network is very different from TCP connection. It reserved fixed amount of bandwidth when circuit is established. Whereas TCP connection will try to transfer data in the shortest time possible and doesn't use any bandwidth when idle 

Ethernet and IP are packet-switched protocols. These protocols do not have the concept of circuit 

Why internet use packet switching? The answer is that they are optimized for *bursty traffic*. 
>A circuit is good for an audio or video call, which needs to transfer a fairly constant number of bits per second for the duration of the call. On the other hand, requesting a web page, sending an email, or transferring a file doesn’t have any particular bandwidth requirement—we just want it to complete as quickly as possible.

If we transfer a file over a circuit, we would have to guess the bandwidth allocation. If we guessed too low, the transfer time will be very long (network capacity is not fully utilized). If we guess too high, the circuit cannot be set up.

using circuits for bursty data transfer wastes network capacity or unnecessarily slow. By contrast, TCP dynamically adapts the rate of data transfer to the available network capacity

>There have been some attempts to build hybrid networks that support both circuit switching and packet switching, such as ATM.iii InfiniBand has some similarities [35]: it implements end-to-end flow control at the link layer, which reduces the need for queueing in the network, although it can still suffer from delays due to link congestion [36]. With careful use of quality of service (QoS, prioritization and scheduling of packets) and admission control (rate-limiting senders), it is possible to emulate circuit switching on packet networks, or provide statistically bounded delay [25, 32].

This sounds promising, but it is not deployed to public internet yet.

>However, such quality of service is currently not enabled in multi-tenant datacenters and public clouds, or when communicating via the internet.iv Currently deployed technology does not allow us to make any guarantees about delays or reliability of the network: we have to assume that network congestion, queueing, and unbounded delays will happen. Consequently, there’s no “correct” value for timeouts—they need to be determined experimentally.


## Unreliable Clocks