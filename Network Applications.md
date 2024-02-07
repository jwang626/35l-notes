# Network Software Construction Theory
#W4L2 #W5L1

Warning- things will move faster, little emphasis on syntax and more on fundamental theory.

## Background

1. What is a network?
	- collection of devices connected to each other to allow sharing of data (e.g. Internet)
	- Key components include 
		- nodes (interconnected devices like computers and servers within network)
		- links (physical or wireless connections that enable data transmission)
		- routers, switches (devices that direct data traffic)
		- protocols (e.g. TCP/IP)
2. What is a network application?
	- software apps that use networks to send/receive data (run on networked computers)
		- e.g. Web Bowsers, email systems, cloud-based services/ Google Drive

### Vocabulary

**Databases** are collections of data that are stored and accessed electronically. In networks, serve as the central repositories where data is stored, retrieved, managed. Essential for network apps that require persistent data storage. Networks typically access data through database servers.

**Servers** are computers or software that provide data, resources, or services to other devices (clients) over a network (process requests and deliver responses). Common server types are web servers, app servers, and database servers.

**Routers** are devices that route data packets from start to endpoint (the switches in packet switching). Job is to connect different segments of a network (traffic control)

## Main Architectural Models

(See picture from lecture)
Network apps can be made from one or more of these models.
Three main architectural frameworks to design and maintain network applications.
1. Peer-to-Peer
2. Primary-Secondary
3. Client-Server (we focus on this one)

### Peer-to-Peer App

Decentralized network. Each node in the network has equal power, acts as both client and server.
- Pros: more robust, if one peer dies other peers will still be able to function. Lower operational cost, easily scalable
- Cons: harder to implement
### Primary-Secondary App

AKA Master-Slave- centralized.
Primary node is "scheduler." Secondary nodes await instructions from primary (worker bees)
- Pros: simpler to implement, better optimization/efficiency (primary node has global picture)
- Cons: not robust (if primary dies)

### Client-Server App

Like a backwards primary-secondary. Single server node + client nodes. However server is still single source of change (all requests go to server).
Clients initiate requests to server, who responds to requests.
- Pros: simplest implementation
- Cons: not robust (if server dies) but most servers have backups. Performance (slower), limited resources/large cost for servers

## Potential issues of Network Apps
- Performance
- Correctness
- Security (solution: encryption)

# Client-Server Software

## Overview of Lecture

Things to understand before talking about Javascript: Infrastructure (includes Internet, WWW, HTML)

Problems of Network Apps (that we didn't need to think about before)
1. Performance: 
	- Latency 
	- Throughput
2. Correctness

## Performance Problems

### Latency

Latency is due to network transmission delay (roundtrip delay).
- Solution: do a simpler approximation of the request AKA client "cheats" by doing stuff *locally* (the amount of cheating depends on purpose of app, i.e. weather app vs bank account)
	- common approach: client caches server data
		- What if you have a **stale cache**? This is a correctness problem...

### Throughput

Throughput is number of problems/sec you can solve. 
- Talk about server throughput (number of requests/sec that server can answer)
How to make server go faster?
- multiple servers (or multiple threads) running in parallel (e.g. by geographical location)
	- note: communication between threads is much easier than between servers
- Front end with parallel back end servers (glue together C-S with P-S architecture)
	- Idea: have parallelization within servers that can do requests & send responses **out of order**
	- out of order could lead to correctness problem (**serialization problems**: do the actions' results make sense?)
As long as we can come up with an *explanation* of the results, this is fine.
- simplest explanation: servers running in parallel but talking to common database that serializes everything (so we consult DB for explanation)
	- downside: bottleneck at DB
	
## Correctness Problems

### Stale Cache

Solution: **cache validation** to ensure data integrity, consistency, and freshness. (Check to make sure cache is valid by seeing if data has been updated on original server.)
- Efficient examples: checksum, timestamp

**Checksum**
A checksum is a small-sized piece of data (digital fingerprint of a set of data) used to identify a block of data. 
- apply algorithm once at beginning, apply at end, if match, then data is valid (unaltered)
- idea of end-to-end checking is crucial for philosophy of Internet infrastructure

**Timestamp**
Applications: version control, concurrency control (keep track of order of operations)

# The Internet: A History

Before the Internet, communicate through network by telephone system, which operated using technique called *circuit switching*

## Circuit switching

Central office has a big switch and a bunch of wires connect switches. A dedicated path is reserved between start and end for the duration of the connection.
*Bandwidth*: max rate at which data can be transferred across a network connection in given time (capacity, not speed).
Pros: 
- once connection established, "guaranteed" performance
Cons:
- cost of maintaining all wires
- wasted capacity when conversation pauses
- "excuse" that led to development of Internet: nuclear attack by Russia on central offices

The idea of packet switching was proposed to solve the issues of circuit switching.

## Packet Switching 

Paul. Baran, a UCLA alum in 1962 (birthplace of Internet)

Break up message into little packets.
Central office might send each packet to destination via different wires (so maybe jumbled order).
Once all packets arrive, reassembled in order.

Application: the Internet is a packet-switched network.

Pros: 
- less wasted capacity through dynamic routing (adaptive paths and load balancing)
- more resilient (decentralized nature- each switch is autonomous, not a single path from start to end)
- scalability (easily add new devices and wires)

### Packets

Recall: a packet is a unit of data that is transmitted over a network. It contains:
1. **Header**: "overhead," control info used by network 
2. **Payload**: the actual data
#### Potential Issues
- packets can be dropped/lost (router overloaded, insufficient buffer space)
- can be received out of order (different routes, or overloaded server)
	- more delay (need to wait for all packets to arrive to render)
- can be duplicated (e.g. network congestion/ retransmission or config issues)

To deal with these issues, write suite of **protocols** (agreements between sender and recipient) so that apps don't have to worry about low level packet issue.

# Protocol Layers

## What are protocols?

A standardized set of rules that define how data is transmitted, received, and interpreted across a network.

We have *layers* of protocols that have distinct functions (which does introduce overhead, but worth it for the functionality)

Below, we will talk about the **TCP/IP** model of network protocols, which has 4 layers.

## Link Layer

The lowest level, deals primarily with hardware from one switch/router to the next
- point to point over links (wires)
- only between 2 adjacent nodes of a network
- Protocols: Ethernet for LANs (local area networks), WiFi for WLANs

## Internet Layer

Think packet switching: this layer is responsible for *individual* packets moving across network from start to endpoint
- Protocols: IP, ICMP

## Transport Layer

No longer a single packet, now talk about data streams/channels (reliable, ordered stream of data packets)
- Protocols: TCP and UDP

## Application Layer

Have protocols designed for a common set of applications to exchange data over network
- focus on Web applications
- Protocols: HTTP, FTP, DNS

# Protocols

## Internet Layer: IP

IPv4: 1983, specified by team led by Postel (UCLA alum)
IPv6: 1996, has 128 bits in IP address/header so slower.
Protocols are documented in Internet RFC (Request For Comments).

IP has to do with contents of packet headers.

The header contains:
- length (number of bytes)
- protocol number
- source & destination address (IP address, 32 bits, territorial)
- TTL (Time-to-Live, to avoid infinite loops)
- simple checksum

Key feature of IP:
- Addressing: each device has unique IP address
- Packet Routing
- each IP packet (called a datagram) is treated independently and IP does not provide reliability at all

## Transport Layer: UDP, TCP

- UDP (User Datagram Protocol, not reliable but fast)
- TCP (Transmission Control Protocol, designed by Cerf, UCLA alum)
### TCP

3 Key Features of TCP data streams:
1. reliable (not lost)
2. ordered 
3. error checked (is a bit slower)

How does TCP do this?
1. establishes *TCP connection* between sender & receiver (three-way handshake)
2. divides stream into packets
3. reassembly (for order)
4. retransmission (for reliability)
5. flow control (don't want to overload router)

#### What is a TCP Connection?

Type of network connection between 2 endpoints over an IP network with the three key features. Crucial for app protocols like HTTP. 

## Application Layer

- HTTP (Hypertext Transfer Protocol, runs atop TCP)
	- need reliability
- RTP (Real-time Transport Protocol, runs atop UDP)
	- need speed, not reliability
	- WebRTP for video streams (e.g. Zoom)
	
### HTTP (for World Wide Web)

HTTP transmits hypermedia documents like HTML on the WWW.
Invented by Tim Berners Lee (1991).
Big idea: *simple protocol* for sharing documents (web pages) and a *simple format* for web pages (haven't talked about the format part yet)

#### HTTP/1.x
- slow, initiated a new TCP connection for each request
- later included persistent connections
- data send in plaintext (not encrypted)

Via TCP: 
Create connection (data channel), send request, get response, close connection.
Client sends server a message: `GET /abc.html HTTP/1.0` 
Uses codes like 200 or 404.

#### HTTP/2
- 2015, developed to improve efficiency
- used header compression

#### HTTPS
Address security concerns of HTTP: supports encryption. The data exchanged between client and server is encrypted.
