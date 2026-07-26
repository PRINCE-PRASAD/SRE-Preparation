# Networking for SRE — Questions and Answers

Plain answers in simple English. Examples and commands included.

---

## Contents

1. [Networking Basics](#networking-basics)
2. [OSI and TCP/IP Models](#osi-and-tcpip-models)
3. [TCP and UDP](#tcp-and-udp)
4. [HTTP and HTTPS](#http-and-https)
5. [DNS](#dns)
6. [Load Balancing](#load-balancing)
7. [Linux Networking](#linux-networking)
8. [Kubernetes Networking](#kubernetes-networking)
9. [AWS Networking](#aws-networking)
10. [Troubleshooting Scenarios](#troubleshooting-scenarios)

---

## Networking Basics

### What is a network?

Two or more devices connected so they can share data. The devices can be connected by cable, Wi-Fi, or over the internet.

Every device needs an address so data knows where to go. That address is the IP address.

### What is an IP address?

A number that identifies a device on a network. It works like a postal address. Data is sent to an IP address, and the network delivers it there.

Example: `192.168.1.10`

An IP address has two parts:

- **Network part** — which network the device is on
- **Host part** — which device on that network

The subnet mask decides where the split is.

### What is the difference between IPv4 and IPv6?

| | IPv4 | IPv6 |
|---|---|---|
| Size | 32 bits | 128 bits |
| Format | Four numbers, dots | Eight groups of hex, colons |
| Example | `192.168.1.10` | `2001:db8::1` |
| Total addresses | About 4.3 billion | An extremely large number |
| NAT needed? | Yes, addresses ran out | No, there are enough |
| Header | More complex | Simpler, faster to process |

IPv6 was created because IPv4 addresses ran out. IPv6 also has built-in support for auto-configuration, so devices can give themselves an address without DHCP.

Short form rules for IPv6: you can remove leading zeros, and replace one run of all-zero groups with `::`.

```
2001:0db8:0000:0000:0000:0000:0000:0001
2001:db8::1                                 (same address, short form)
```

### What is a subnet?

A smaller network inside a bigger network.

Instead of putting 10,000 devices on one network, you split them into smaller groups. Reasons to do this:

- **Less broadcast traffic.** Broadcasts stay inside one subnet.
- **Security.** You can put rules between subnets, for example web servers in one subnet and databases in another.
- **Organisation.** Different subnets for different teams, or different availability zones.

### What is a subnet mask?

A number that says which part of an IP address is the network, and which part is the device.

```
IP address:    192.168.1.10
Subnet mask:   255.255.255.0
               ^^^^^^^^^^^ ^
               network     device
```

So the network is `192.168.1`, and `.10` is the device.

Common masks:

| Subnet mask | CIDR | Total addresses |
|---|---|---|
| 255.0.0.0 | /8 | 16,777,216 |
| 255.255.0.0 | /16 | 65,536 |
| 255.255.255.0 | /24 | 256 |
| 255.255.255.128 | /25 | 128 |
| 255.255.255.192 | /26 | 64 |
| 255.255.255.240 | /28 | 16 |

### What is CIDR notation?

A short way to write a subnet mask. You write a slash and the number of network bits.

```
192.168.1.0/24     means mask 255.255.255.0
10.0.0.0/16        means mask 255.255.0.0
```

**How to work out the size:** an IPv4 address has 32 bits. Subtract the CIDR number, then use that as a power of 2.

```
/24  →  32 − 24 = 8   →  2^8  = 256 addresses
/26  →  32 − 26 = 6   →  2^6  = 64 addresses
/28  →  32 − 28 = 4   →  2^4  = 16 addresses
```

A smaller number after the slash means a bigger network.

**Usable addresses are fewer.** Normally two addresses are reserved in each subnet: the first (network address) and the last (broadcast address). So a `/24` gives 254 usable addresses.

**On AWS, five are reserved**, not two. In a `/24` you get 251 usable addresses. AWS reserves:

- `.0` — network address
- `.1` — the VPC router
- `.2` — DNS
- `.3` — reserved for future use
- `.255` — broadcast address

### What is a gateway?

The device that sends your traffic out of your local network.

When your computer wants to reach an IP address that is **not** in its own subnet, it does not know the route. So it sends the packet to the gateway, and the gateway forwards it onward.

This is why it is often called the "default gateway" — it is the default place to send anything you do not have a specific route for.

```bash
ip route | grep default
# default via 192.168.1.1 dev eth0
```

Here `192.168.1.1` is the gateway.

### What is a MAC address?

A hardware address burned into a network card. It is 48 bits, written in hex.

Example: `00:1A:2B:3C:4D:5E`

The difference from an IP address:

| | MAC address | IP address |
|---|---|---|
| Set by | The hardware maker | The network / DHCP |
| Changes | Usually never | Changes when you move network |
| Works | Only on the local network | Across the whole internet |
| Layer | Layer 2 | Layer 3 |

A MAC address only works inside one local network. Routers do not forward MAC addresses. Each time a packet moves to a new network segment, the MAC addresses are replaced, but the IP addresses stay the same.

### What is ARP?

ARP means Address Resolution Protocol. It finds the MAC address that belongs to an IP address, on the local network.

Why it is needed: to actually send a packet on the local network, the device needs the destination MAC address. It only has the IP address. ARP fills that gap.

How it works:

1. The device shouts to everyone on the network: "Who has `192.168.1.10`?"
2. The device with that IP replies: "That is me, my MAC is `00:1A:2B:3C:4D:5E`."
3. The answer is saved in the ARP cache, so it does not need to ask again.

```bash
ip neigh          # show the ARP cache
arp -n            # older command, same idea
```

### What is the difference between a public IP and a private IP?

**Public IP** — unique on the whole internet. Anyone can route to it.

**Private IP** — only used inside a private network. Not routable on the internet. Many companies use the same private ranges at the same time, and that is fine because they never meet.

The private ranges are fixed:

| Range | CIDR | Size |
|---|---|---|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | 16.7 million |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | 1 million |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | 65,536 |

Private IPs exist because IPv4 addresses ran out. Instead of giving a public IP to every device, a company uses private IPs inside and shares a small number of public IPs to reach the internet. NAT makes that work.

### What is NAT?

NAT means Network Address Translation. It changes the IP address in a packet as it passes through a router.

The common use: many devices with private IPs share one public IP to reach the internet.

```
Your laptop (10.0.0.5)  →  NAT router  →  Internet
source IP: 10.0.0.5        source IP changed to: 52.1.2.3
```

The reply comes back to `52.1.2.3`, and the router remembers to send it to `10.0.0.5`.

Why NAT matters: it saves public IP addresses, and it hides internal addresses from the outside. A side effect is that outside devices cannot start a connection to your private device, which is a basic form of protection.

### What is PAT?

PAT means Port Address Translation. It is NAT plus port numbers. It is also called NAT overload, and it is the type of NAT almost everyone actually uses.

Plain NAT maps one private IP to one public IP. That does not help much when you have 500 devices.

PAT solves this by also changing the **port** number, so many devices can share **one** public IP:

| Private source | Public source after PAT |
|---|---|
| 10.0.0.5:41000 | 52.1.2.3:**52001** |
| 10.0.0.6:41000 | 52.1.2.3:**52002** |
| 10.0.0.7:38000 | 52.1.2.3:**52003** |

The router keeps a table of these mappings. When a reply arrives on port 52002, the router knows it belongs to `10.0.0.6`.

**The limit to know:** one public IP has about 65,000 ports. So one IP can hold roughly 65,000 connections **to the same destination IP and port**. AWS NAT Gateway has this limit, and it shows up as the `ErrorPortAllocation` metric when you run out.

### What is DNS?

DNS means Domain Name System. It turns names into IP addresses.

People remember `google.com`. Computers need `142.250.185.78`. DNS does the translation.

It also does more than that. It stores mail server addresses, text records for verification, and other information about a domain.

### How does DNS resolution work?

Say you look up `www.example.com`. If nothing is cached, this happens:

```
1. Your computer checks its own cache.           Found? Done.
2. It asks the resolver (your ISP, or 8.8.8.8).  Cached? Done.
3. The resolver asks a ROOT server.
   Root says: "I do not know, but ask the .com servers."
4. The resolver asks a .com TLD server.
   TLD says: "I do not know, but ask example.com's nameservers."
5. The resolver asks example.com's AUTHORITATIVE nameserver.
   That server says: "www.example.com is 93.184.216.34."
6. The resolver saves the answer in its cache, then gives it to you.
```

Two important terms:

- **Recursive resolver** — the server that does all this work for you and gives you one final answer. Your ISP resolver or `8.8.8.8`.
- **Authoritative nameserver** — the server that actually holds the real records for a domain. It is the source of truth.

You can watch these steps yourself:

```bash
dig +trace www.example.com
```

### What is DHCP?

DHCP means Dynamic Host Configuration Protocol. It gives a device its network settings automatically, so nobody has to type them by hand.

It gives out:

- An IP address
- A subnet mask
- The default gateway
- DNS server addresses

The four steps are called **DORA**:

| Step | What happens |
|---|---|
| **D**iscover | The device shouts: "Is there a DHCP server?" |
| **O**ffer | A server replies: "Yes, you can use 192.168.1.50." |
| **R**equest | The device says: "I accept that address." |
| **A**cknowledge | The server confirms and records the lease. |

Addresses are given out as a **lease** with a time limit. Before the lease runs out, the device asks to renew it.

### What is ICMP?

ICMP means Internet Control Message Protocol. It carries messages **about** the network, not user data.

It has no ports, and it is not TCP or UDP. It is its own protocol.

Common uses:

| ICMP message | Used by |
|---|---|
| Echo Request / Echo Reply | `ping` |
| Time Exceeded | `traceroute` |
| Destination Unreachable | Tells you a route or port is not available |
| Fragmentation Needed | Used by MTU discovery |

**Important thing to know:** firewalls often block ICMP. So `ping` failing does **not** mean the server is down. It may just mean ICMP is blocked. Always test the real port instead:

```bash
nc -zv example.com 443
```

### What is MTU?

MTU means Maximum Transmission Unit. It is the largest packet size a network link can carry, in bytes.

The normal value on Ethernet is **1500 bytes**. Jumbo frames are **9000 bytes**, used inside data centres. Inside AWS, between instances in the same VPC, 9001 is supported.

If a packet is bigger than the MTU, it must be split up, or dropped.

**Why this causes strange problems:** when the MTU is wrong somewhere in the path, small packets work fine and large packets fail. So a `ping` works, a small API call works, but a large file upload hangs. That specific pattern almost always means an MTU problem.

```bash
ip link show eth0                    # see the MTU
ping -M do -s 1472 example.com       # test 1500 MTU (1472 + 28 bytes of headers)
```

`-M do` means "do not fragment". If that fails but a smaller size works, you have found the real MTU limit.

### What is fragmentation?

Splitting one packet into smaller pieces, because it is bigger than the MTU of the link.

The receiver puts the pieces back together.

Why it is a problem:

- It is slow. More packets, more work.
- If **one** piece is lost, the **whole** packet is lost and must be sent again.
- Some firewalls drop fragments.

**In IPv6, routers do not fragment at all.** If a packet is too big, the router drops it and sends back an ICMP message saying "too big". The sender must then use smaller packets. This process is called Path MTU Discovery.

This is why blocking all ICMP can break things badly. If those "too big" messages are blocked, the sender never learns, and the connection just hangs. This is called an MTU black hole.

### What is TTL?

There are two different meanings. Do not mix them up.

**1. TTL in an IP packet (hop limit)**

A number in the packet header. Every router it passes through subtracts 1. When it reaches 0, the router throws the packet away and sends back an ICMP "Time Exceeded" message.

Its job is to stop packets circling forever if there is a routing loop.

Starting values are usually 64 (Linux), 128 (Windows), or 255.

`traceroute` uses this on purpose. It sends a packet with TTL 1, and the first router replies. Then TTL 2, and the second router replies. That is how it maps the path.

**2. TTL in DNS**

A time in seconds. It says how long a DNS answer may be cached before asking again.

```
www.example.com.   300   IN   A   93.184.216.34
                   ^^^
                   TTL = 300 seconds = 5 minutes
```

A low TTL means changes spread quickly, but there are more DNS lookups. A high TTL means less DNS traffic, but changes take longer to reach everyone.

**Useful tip:** lower the TTL **before** you plan to change a record. If you lower it afterwards, it is too late — the old high TTL is already cached everywhere.

---

## OSI and TCP/IP Models

### Explain the OSI model.

A model with 7 layers that describes how network communication works, step by step. Each layer has one job and uses the layer below it.

Read it from the bottom up:

| Layer | Name | Job | Examples |
|---|---|---|---|
| 7 | Application | What the user's program does | HTTP, DNS, SSH, SMTP |
| 6 | Presentation | Format, encrypt, compress | TLS, JPEG, ASCII |
| 5 | Session | Start, manage and end sessions | Sockets, RPC |
| 4 | Transport | Deliver to the right program, reliability | **TCP, UDP** |
| 3 | Network | Find the path between networks | **IP**, ICMP, routers |
| 2 | Data Link | Move data on one local link | **MAC**, Ethernet, ARP, switches |
| 1 | Physical | The actual signal | Cables, Wi-Fi radio, fibre |

A simple way to remember the useful part:

- **Layer 2** uses MAC addresses. Works inside one local network. Switches.
- **Layer 3** uses IP addresses. Works between networks. Routers.
- **Layer 4** uses ports. Decides which program gets the data. TCP and UDP.
- **Layer 7** is the actual content. HTTP requests, DNS queries.

Layers 4 and 7 matter most day to day, because load balancers are described as "layer 4" or "layer 7".

### Explain the TCP/IP model.

A simpler model with 4 layers. This is the one the real internet is built on. OSI is a teaching model; TCP/IP is what is actually used.

| TCP/IP layer | Job | Same as OSI layers |
|---|---|---|
| Application | Everything the app does, including encryption | 5, 6, 7 |
| Transport | TCP, UDP | 4 |
| Internet | IP, ICMP, routing | 3 |
| Link (Network Access) | Ethernet, Wi-Fi, MAC, physical | 1, 2 |

### What is the difference between OSI and TCP/IP?

| | OSI | TCP/IP |
|---|---|---|
| Layers | 7 | 4 |
| Purpose | A teaching and reference model | The model actually in use |
| Created | By a standards committee, before the protocols | Built from protocols that already worked |
| Top layers | Splits application into 3 layers | Combines them into 1 |
| Used in practice | For talking about layers | For real implementation |

They describe the same thing at different levels of detail. People use OSI numbers in conversation ("that is a layer 7 problem") but use TCP/IP in real systems.

### Which protocol works at each OSI layer?

| Layer | Protocols |
|---|---|
| 7 Application | HTTP, HTTPS, DNS, SSH, FTP, SMTP, SNMP, DHCP |
| 6 Presentation | TLS/SSL, JPEG, PNG, ASCII, UTF-8 |
| 5 Session | NetBIOS, RPC, SOCKS |
| 4 Transport | TCP, UDP, QUIC, SCTP |
| 3 Network | IP, ICMP, IGMP, IPsec, OSPF, BGP |
| 2 Data Link | Ethernet, ARP, PPP, VLAN (802.1Q), Wi-Fi (802.11) |
| 1 Physical | Cables, fibre, radio signals, connectors, voltages |

Note that some protocols do not fit neatly. ARP sits between layers 2 and 3. BGP runs on top of TCP but does layer 3 work. The model is a guide, not a strict rule.

### Where does TLS work in the OSI model?

TLS sits **above TCP** and **below the application**. So the honest answer is: between layer 4 and layer 7.

If you must give one number, most people say **layer 6 (Presentation)**, because encryption is a presentation-layer job. Some say layer 5.

What is more useful than the number is the order of events:

```
1. TCP connection opens        (layer 4 — three-way handshake)
2. TLS handshake happens       (certificates checked, keys agreed)
3. HTTP request is sent        (layer 7 — now encrypted)
```

This order explains real problems:

- If the TCP connection fails, you never reach TLS. The error is "connection refused" or "timeout".
- If TLS fails, TCP worked fine. The error is about certificates.
- If the HTTP request fails, both TCP and TLS worked. The error is a status code like 500.

So the error message tells you which stage broke.

---

## TCP and UDP

### What is the difference between TCP and UDP?

Both deliver data to a program using ports. The difference is whether they check that the data arrived.

| | TCP | UDP |
|---|---|---|
| Connection | Yes, set up first | No, just send |
| Reliable | Yes, resends lost data | No, lost data stays lost |
| Order | Data arrives in order | May arrive out of order |
| Speed | Slower (more checks) | Faster (no checks) |
| Header size | 20 bytes | 8 bytes |
| Flow control | Yes | No |
| Used by | HTTP, SSH, databases, email | DNS, video streaming, games, VoIP |

**How to choose:** if losing data breaks things, use TCP. If speed matters more than perfection, use UDP.

A video call is a good example. If one frame is lost, you do not want the whole call to pause and wait for it. You want to keep going. So UDP is correct there.

**Note:** DNS mostly uses UDP, because a query is small and fast. But it switches to TCP when the answer is too large (over 512 bytes), or for zone transfers.

### Explain the TCP three-way handshake.

The three messages that open a TCP connection.

```
Client                          Server
  |                               |
  |------- 1. SYN --------------->|   "I want to connect. My sequence number is X."
  |                               |
  |<------ 2. SYN-ACK ------------|   "OK. My number is Y. I got your X."
  |                               |
  |------- 3. ACK --------------->|   "I got your Y. Connection is open."
  |                               |
  |====== connection ready =======|
```

Why three messages and not two? Because **both** sides need to confirm they can send and receive. Each side has to prove it received the other side's starting number.

**Why this matters in practice:** the handshake costs one full round trip before any data moves. If the server is 200ms away, you pay 200ms before sending your first byte. Then TLS adds more round trips. This is why connection reuse (keep-alive) makes such a big difference to speed.

**What failures look like:**

| Symptom | Meaning |
|---|---|
| Connection refused (instantly) | The server is reachable, but nothing is listening on that port |
| Timeout (hangs, then fails) | Nothing replied at all — usually a firewall dropping the packet silently |

That difference is very useful. "Refused" means you reached the machine. "Timeout" means you did not.

### Explain the TCP four-way termination.

The messages that close a TCP connection properly. It takes four, because each direction is closed separately.

```
Client                          Server
  |------- 1. FIN -------------->|   "I have no more data to send."
  |<------ 2. ACK --------------|   "OK, I know."
  |                              |
  |     (server can still send)  |
  |                              |
  |<------ 3. FIN --------------|   "Now I have no more data either."
  |------- 4. ACK ------------->|   "OK. Closed."
```

Why four? Because TCP closes each direction on its own. One side can stop sending while still receiving. That is called a half-close.

**TIME_WAIT.** After closing, the side that closed first stays in a state called `TIME_WAIT` for about 60 seconds on Linux. It waits in case the last ACK was lost, and to stop old packets confusing a new connection using the same ports.

You may see thousands of connections in `TIME_WAIT` on a busy server:

```bash
ss -tan | awk '{print $1}' | sort | uniq -c
```

Many `TIME_WAIT` entries are normal, not a problem by themselves. But if you run out of ports, the fix is connection reuse (keep-alive), not changing kernel settings.

Note: `RST` (reset) is different. It closes a connection immediately with no handshake. It usually means an error, a closed port, or a firewall cutting the connection.

### What is a TCP connection?

A two-way channel between two programs, identified by four things together:

```
source IP  +  source port  +  destination IP  +  destination port
```

This group of four is called the **4-tuple**. It must be unique. If any one value is different, it is a different connection.

This is how one server can hold many connections on port 443. Every client has a different IP or port, so each 4-tuple is unique.

```
Client A: 1.1.1.1:50001  →  server 5.5.5.5:443    ← connection 1
Client A: 1.1.1.1:50002  →  server 5.5.5.5:443    ← connection 2 (different source port)
Client B: 2.2.2.2:50001  →  server 5.5.5.5:443    ← connection 3 (different source IP)
```

A TCP connection also keeps state: sequence numbers, window sizes, and its current state (`ESTABLISHED`, `TIME_WAIT`, and so on).

### What is a socket?

A socket is the thing your program uses to send and receive network data. The operating system gives it to you, and it looks like a file to the program.

Two ways the word is used:

- **A listening socket** — an IP and port the server waits on. For example `0.0.0.0:443`.
- **A connected socket** — one end of an actual connection, so it holds the full 4-tuple.

A rough picture: a port is the address, and a socket is the open door at that address.

```bash
ss -tulnp        # show all sockets: listening and connected
```

### What is a port?

A number that says which program on the machine should get the data.

An IP address finds the **machine**. A port finds the **program** on that machine.

Ports go from 0 to 65535, in three groups:

| Range | Name | Used for |
|---|---|---|
| 0 – 1023 | Well-known | Standard services. Needs admin rights to use. |
| 1024 – 49151 | Registered | Applications that registered a port |
| 49152 – 65535 | Dynamic / ephemeral | Temporary ports for outgoing connections |

Common ports worth knowing:

| Port | Service |
|---|---|
| 22 | SSH |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 27017 | MongoDB |
| 9090 | Prometheus |
| 2379 / 2380 | etcd |

### What are ephemeral ports?

Short-lived ports the operating system picks automatically for **outgoing** connections.

When your program connects to a web server, it needs a source port. It does not care which one. So the kernel picks a free port from the ephemeral range.

Check the range on Linux:

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
# 32768   60999
```

That gives about 28,000 ports by default.

**Why an SRE cares:** you can run out of them. This happens when a server makes very many outgoing connections, for example a proxy or an app calling a database.

Symptoms: connections start failing with "cannot assign requested address", even though CPU and memory are fine.

Fixes, in order of preference:

1. **Use keep-alive** and reuse connections instead of opening a new one every time. This is the real fix.
2. Widen the ephemeral range.
3. Enable `tcp_tw_reuse` so `TIME_WAIT` ports can be reused sooner.

The same limit applies to NAT gateways, because all traffic shares one public IP and one port range.

### What causes TCP retransmissions?

A retransmission means the sender did not get an ACK in time, so it sent the data again.

Causes:

| Cause | Explanation |
|---|---|
| Packet loss | A router dropped the packet, often because its queue was full |
| Congestion | The network is overloaded, so packets are dropped |
| Bad hardware | A failing cable, port, or network card |
| MTU problems | Packets too big are dropped and never arrive |
| Receiver overloaded | The server is too busy to read from its buffers |
| Firewall dropping traffic | Silently thrown away, no reply sent |
| Wi-Fi interference | Common on wireless links |

Some retransmissions are completely normal — TCP is designed for this. A small percentage is fine. A high and rising rate means a real problem.

How to see them:

```bash
ss -ti                                  # per-connection stats, look for "retrans"
netstat -s | grep -i retrans            # totals for the system
tcpdump -i eth0 'tcp[tcpflags] & tcp-push != 0'
```

The effect on users: retransmissions make latency jump, because TCP waits before resending. That is why "the network is slow" often turns out to be packet loss, not low bandwidth.

### What is TCP Keepalive?

A feature that sends small probe packets on an idle connection, to check the other side is still there.

Why it is needed: TCP has no way to know if the other machine crashed or was unplugged. Without keepalive, a dead connection can sit there forever using memory, and the app thinks it is fine.

Default settings on Linux are very slow:

```bash
sysctl net.ipv4.tcp_keepalive_time     # 7200  (2 hours before the first probe)
sysctl net.ipv4.tcp_keepalive_intvl    # 75    (seconds between probes)
sysctl net.ipv4.tcp_keepalive_probes   # 9     (probes before giving up)
```

Two hours is far too long for most systems. This causes a real and common problem:

A load balancer or NAT gateway drops idle connections after a few minutes. Your app does not know. It tries to use the connection much later and hangs. **The fix is to set keepalive lower than the load balancer's idle timeout.** AWS NLB, for example, has a 350-second idle timeout, so keepalive should be under that.

**Note:** TCP keepalive is different from HTTP keep-alive. TCP keepalive checks if a connection is alive. HTTP keep-alive reuses a connection for several requests so you do not pay for a new handshake each time.

### What is Window Size?

How much data the receiver can accept before it must send an ACK. It is flow control — it stops a fast sender from flooding a slow receiver.

The receiver says in every packet: "I have room for N more bytes." The sender must not send more than that without waiting.

**Why it limits speed.** The maximum speed you can get is:

```
throughput = window size ÷ round trip time
```

So on a long-distance link, a small window makes everything slow, no matter how much bandwidth you have. A 64KB window over a 100ms link gives you only about 5 Mbit/s, even on a 10 Gbit connection.

**Window scaling** fixes this. The original TCP window field is only 16 bits, so the maximum is 64KB. Window scaling multiplies it, allowing much larger windows. It is on by default in modern Linux.

If you see a connection that is slow but has no packet loss, and it is over a long distance, the window is the usual suspect.

### What is Congestion Control?

TCP's way of finding the right sending speed, so it does not overload the network.

The idea: the sender does not know how much bandwidth is available. So it starts slow, speeds up while things work, and slows down when it sees packet loss.

The main phases:

| Phase | What happens |
|---|---|
| **Slow start** | Start small, double the rate each round trip |
| **Congestion avoidance** | After a threshold, grow slowly (add a little each round trip) |
| **Fast retransmit** | Resend a lost packet quickly, without waiting for a full timeout |
| **Fast recovery** | Cut the rate, but do not go all the way back to the start |

The algorithms you may hear about:

- **Reno** — the classic one. Cuts speed in half on any loss.
- **CUBIC** — the default in Linux. Recovers faster on high-speed, long-distance links.
- **BBR** — from Google. Measures bandwidth and delay instead of reacting to loss. Much better on networks that lose packets for reasons other than congestion, such as mobile.

**Why this matters for SRE:** congestion control treats packet loss as a signal to slow down. So a small amount of packet loss reduces throughput a lot. This is why a link with 2% loss can feel completely broken even though 98% of packets arrive.

Slow start also explains why short connections are slow. A connection that only sends a small amount of data never gets up to full speed. Again, connection reuse helps.

---

## HTTP and HTTPS

### What is HTTP?

HyperText Transfer Protocol. The rules a browser and a web server use to talk. It runs on TCP, usually on port 80.

It is a request-and-response protocol. The client sends a request. The server sends a response. Then it is finished.

A simple request looks like this:

```http
GET /index.html HTTP/1.1
Host: example.com
User-Agent: curl/8.0
```

And a response:

```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1256

<html>...</html>
```

HTTP is **stateless**. The server does not remember your last request. Cookies and tokens are used to add memory on top.

Versions:

| Version | Key point |
|---|---|
| HTTP/1.1 | One request at a time per connection. Text based. |
| HTTP/2 | Many requests at once on one connection. Binary. Header compression. |
| HTTP/3 | Runs on QUIC (which uses UDP) instead of TCP. Faster to set up and better on poor networks. |

### What is HTTPS?

HTTP with TLS encryption. It runs on port 443.

It is exactly the same HTTP protocol. The difference is that TLS wraps it, so the traffic is encrypted before it leaves your machine.

HTTPS gives three things:

1. **Encryption** — nobody in the middle can read the data
2. **Authentication** — the certificate proves the server is really that site
3. **Integrity** — nobody can change the data without being detected

Point 2 matters as much as point 1. Encryption alone is not enough. Without a certificate check, you could be talking securely to an attacker.

### What is the difference between HTTP and HTTPS?

| | HTTP | HTTPS |
|---|---|---|
| Port | 80 | 443 |
| Encrypted | No | Yes |
| Certificate needed | No | Yes |
| Setup speed | One round trip (TCP) | Extra round trips for TLS |
| Browser shows | "Not secure" | Padlock |
| SEO | Worse | Better |
| HTTP/2 support | In practice, no | Yes |

The speed difference is now very small. TLS 1.3 needs only one extra round trip, and session resumption can reduce that to zero. Modern hardware handles the encryption easily. There is no good reason to use plain HTTP for a public site.

### What happens when you enter a URL in a browser?

The full sequence:

```
1.  Parse the URL
    Split it into scheme (https), host (example.com), port (443), path (/page)

2.  Check the HSTS list
    If the site requires HTTPS, upgrade http:// to https:// before sending anything

3.  DNS lookup
    Browser cache → OS cache → resolver → root → TLD → authoritative
    Result: an IP address

4.  Open a TCP connection
    Three-way handshake: SYN, SYN-ACK, ACK

5.  TLS handshake (for HTTPS)
    ClientHello, ServerHello, certificate, key agreement, Finished
    The browser checks the certificate: correct name, not expired, trusted issuer

6.  Send the HTTP request
    GET /page HTTP/1.1 with headers and cookies

7.  Server processes it
    Often: load balancer → app server → database → response

8.  Receive the HTTP response
    Status code, headers, body

9.  Browser renders the page
    Parse HTML → build the DOM → request CSS, JS and images (more of steps 3 to 8)
    → run JavaScript → paint the page

10. Keep the connection open
    Keep-alive means the next request skips steps 4 and 5
```

This sequence is useful for debugging, because you can test each step on its own:

```bash
dig example.com                              # step 3
nc -zv example.com 443                       # step 4
openssl s_client -connect example.com:443    # step 5
curl -v https://example.com                  # steps 6 to 8
```

### What are HTTP methods?

The verb in a request. It says what you want to do.

| Method | Purpose | Safe? | Idempotent? |
|---|---|---|---|
| GET | Read something | Yes | Yes |
| HEAD | Like GET, but headers only, no body | Yes | Yes |
| POST | Create something, or send data | No | **No** |
| PUT | Replace something completely | No | Yes |
| PATCH | Change part of something | No | No |
| DELETE | Remove something | No | Yes |
| OPTIONS | Ask what is allowed (used by CORS) | Yes | Yes |

Two words to understand:

- **Safe** means it does not change anything on the server.
- **Idempotent** means doing it 10 times gives the same result as doing it once.

**Why idempotent matters for SRE:** it decides whether a retry is safe. You can safely retry a GET, PUT, or DELETE. Retrying a POST may create two records or charge a card twice. This is why retry rules in load balancers and clients treat POST differently.

### What is the difference between PUT and POST?

| | POST | PUT |
|---|---|---|
| Meaning | Create a new thing | Replace the thing at this exact address |
| Where | Sent to a collection: `/users` | Sent to one item: `/users/123` |
| Server picks the ID | Yes | No, you provide it |
| Idempotent | No | Yes |
| Run it twice | Creates two records | Same result both times |

```http
POST /users          →  creates /users/456, server chose the ID
PUT  /users/456      →  replaces user 456 with exactly this data
```

Run POST twice and you get two users. Run PUT twice and user 456 just ends up in the same state. That is the whole difference.

`PATCH` is for partial changes — send only the fields you want to change, rather than the whole object.

### What are HTTP status codes?

A three-digit number in the response saying what happened. The first digit gives the category.

| Range | Meaning |
|---|---|
| **1xx** | Information. Still processing. |
| **2xx** | Success. |
| **3xx** | Redirect. Look somewhere else. |
| **4xx** | Client error. The request was wrong. |
| **5xx** | Server error. The server failed. |

The ones you actually see:

| Code | Name | Meaning |
|---|---|---|
| 200 | OK | Worked |
| 201 | Created | Created successfully, usually after POST |
| 204 | No Content | Worked, nothing to return |
| 301 | Moved Permanently | Permanent redirect |
| 302 | Found | Temporary redirect |
| 304 | Not Modified | Your cached copy is still good |
| 400 | Bad Request | The request itself is malformed |
| 401 | Unauthorized | You are not logged in |
| 403 | Forbidden | You are logged in but not allowed |
| 404 | Not Found | No such thing |
| 405 | Method Not Allowed | Wrong verb for this path |
| 429 | Too Many Requests | Rate limited |
| 499 | Client Closed Request | NGINX only — the client gave up first |
| 500 | Internal Server Error | The app crashed or threw an error |
| 502 | Bad Gateway | The proxy got an invalid answer from the app |
| 503 | Service Unavailable | Overloaded, or no healthy backend |
| 504 | Gateway Timeout | The app was too slow to answer |

**The 4xx and 5xx split is important.** 4xx means the client's fault, so retrying the same request will not help. 5xx means the server's fault, so a retry may work. This is why monitoring should track them separately — a jump in 4xx is often a broken client or a bad deploy of a frontend, while a jump in 5xx is your problem.

### What is a redirect, and what is the difference between 301 and 302?

A redirect tells the client "this thing is at a different address, go there instead". The new address is in the `Location` header.

| | 301 | 302 |
|---|---|---|
| Meaning | Moved permanently | Moved temporarily |
| Browser caching | Cached, often for a long time | Not cached by default |
| Search engines | Move ranking to the new URL | Keep ranking on the old URL |
| Use for | A permanent URL change, http→https | A/B tests, temporary maintenance page, login redirect |

**A serious warning about 301:** browsers cache it aggressively, sometimes forever. If you send a 301 to the wrong place by mistake, users stay stuck on the wrong page even after you fix the server. They must clear their browser cache.

So when in doubt, use 302. Only use 301 when you are certain the change is permanent.

Related codes: `307` and `308` are the same as 302 and 301, but they promise not to change the HTTP method. A 301 or 302 may turn a POST into a GET; 307 and 308 will not.

### What is SSL/TLS?

The protocol that encrypts network traffic.

SSL is the old name. All SSL versions are broken and should never be used. TLS is the current name. People still say "SSL" out of habit, but they mean TLS.

Versions:

| Version | Status |
|---|---|
| SSL 2.0, SSL 3.0 | Broken. Disabled. |
| TLS 1.0, TLS 1.1 | Old and insecure. Removed from browsers. |
| **TLS 1.2** | Still fine and widely used |
| **TLS 1.3** | Current best. Faster and simpler. |

TLS 1.3 is better in two ways: it needs fewer round trips to set up, and it removed all the old weak options that caused most TLS security problems.

### Explain the TLS handshake.

The steps where the two sides agree on encryption keys and check the certificate.

**TLS 1.2** (two round trips):

```
Client                                    Server
  |---- ClientHello ---------------------->|   TLS versions and ciphers I support,
  |                                        |   plus a random number
  |<--- ServerHello ----------------------|   Chosen version and cipher, my random number
  |<--- Certificate ----------------------|   My certificate, with my public key
  |<--- ServerHelloDone ------------------|
  |---- ClientKeyExchange --------------->|   Key material, encrypted with the public key
  |---- ChangeCipherSpec / Finished ----->|   From now on, everything is encrypted
  |<--- ChangeCipherSpec / Finished ------|
  |========= encrypted data ==============|
```

**TLS 1.3** (one round trip). The client guesses which key agreement method the server will pick and sends its key share in the first message. So the handshake finishes in half the time.

**What the client checks on the certificate:**

1. Is the name correct? Does it match the hostname you asked for?
2. Is it inside its valid dates? Not expired, not too early.
3. Is it signed by a trusted authority? Can it build a chain up to a root the client trusts?
4. Is it revoked? Checked with OCSP or CRL.

If any check fails, the browser shows a warning. Each failure gives a **different** error message, which is very useful for debugging:

| Error | Which check failed |
|---|---|
| `ERR_CERT_DATE_INVALID` | Dates — expired |
| `ERR_CERT_COMMON_NAME_INVALID` | Name does not match |
| `ERR_CERT_AUTHORITY_INVALID` | Chain is not trusted, often a missing intermediate certificate |

### What is a certificate?

A file that proves a server owns a domain name. It contains:

- The domain name it is valid for (the Common Name, and the Subject Alternative Names)
- The server's public key
- Valid from and valid until dates
- Who issued it (the Certificate Authority)
- The CA's digital signature

The trust works as a chain:

```
Root CA           (already trusted by your operating system and browser)
   ↓ signs
Intermediate CA
   ↓ signs
Your certificate  (example.com)
```

The client follows the chain upward until it reaches a root it already trusts.

**The most common real problem** is a missing intermediate certificate. The server sends only its own certificate and forgets the intermediate. Then the client cannot complete the chain. It often works in a browser (browsers can sometimes fetch the missing part) but fails in `curl`, in Java, and in mobile apps. That is a confusing bug, and this is the cause.

Inspect a certificate:

```bash
openssl s_client -connect example.com:443 -servername example.com
openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -dates -subject -issuer
```

### What is SNI?

SNI means Server Name Indication. The client puts the hostname it wants **inside the first TLS message**.

Why it is needed: TLS starts before HTTP. The server must choose a certificate before it can see the HTTP `Host` header. Without SNI, one IP address could serve only one HTTPS site, because the server would not know which certificate to send.

With SNI, one IP can serve thousands of HTTPS sites. This is how shared hosting and CDNs work.

```bash
# Without -servername, no SNI is sent — you may get the wrong certificate
openssl s_client -connect example.com:443 -servername example.com
```

Two practical notes:

- **SNI is not encrypted** in TLS 1.2 or 1.3. Anyone watching can see which site you are visiting, even though the content is encrypted. A newer feature called Encrypted Client Hello (ECH) fixes this, but it is not widely used yet.
- Very old clients do not support SNI. They receive the server's default certificate, which is usually the wrong one, so they get a name mismatch error.

---

## DNS

### What is an A record?

Maps a name to an **IPv4 address**.

```
example.com.        300   IN   A      93.184.216.34
```

The `AAAA` record does the same for IPv6:

```
example.com.        300   IN   AAAA   2606:2800:220:1:248:1893:25c8:1946
```

One name can have several A records. The resolver returns all of them, and the client picks one. This is a simple form of load balancing, but it has no health checking — if one IP is dead, some users still get it.

### What is a CNAME?

An alias. It points one name at **another name**, not at an IP address.

```
www.example.com.    300   IN   CNAME   example.com.
```

When a client looks up `www.example.com`, it gets told "look up `example.com` instead", then looks that up too. So a CNAME costs an extra lookup.

**Two important rules:**

1. **You cannot put a CNAME at the zone apex.** The apex is the bare domain, `example.com` with no subdomain. This is not allowed because the apex must also hold NS and SOA records, and a CNAME cannot exist next to other records.

   This is a real problem when you want `example.com` to point at a load balancer, which only gives you a DNS name. The solution on AWS is a **Route 53 alias record**, which behaves like a CNAME but is allowed at the apex. Other providers call it ANAME or flattened CNAME.

2. **A CNAME cannot have other records alongside it** for the same name.

### What is an MX record?

Says which mail servers receive email for a domain. It includes a priority number.

```
example.com.   3600   IN   MX   10   mail1.example.com.
example.com.   3600   IN   MX   20   mail2.example.com.
```

**Lower number means higher priority.** So mail goes to `mail1` first. If that fails, it tries `mail2`. This gives mail redundancy.

An MX record must point to a **name**, never to an IP address.

### What is a TXT record?

Holds free text. It was originally for notes, but now it is used for machine-readable settings.

Main uses today:

| Use | Example |
|---|---|
| **SPF** — which servers may send mail as you | `v=spf1 include:_spf.google.com ~all` |
| **DKIM** — public key for signing mail | `v=DKIM1; k=rsa; p=MIGfMA0...` |
| **DMARC** — what to do with mail that fails checks | `v=DMARC1; p=reject; rua=mailto:...` |
| **Domain verification** | `google-site-verification=abc123` |
| **ACME / certificate validation** | `_acme-challenge.example.com` |

The certificate one matters for SRE. Let's Encrypt DNS validation and AWS ACM validation both work by checking a DNS record. If someone deletes that record later, certificate renewal fails silently.

### What is an NS record?

Says which nameservers are authoritative for a domain — that is, which servers hold the real records.

```
example.com.   172800   IN   NS   ns1.example-dns.com.
example.com.   172800   IN   NS   ns2.example-dns.com.
```

This is how DNS delegation works. The `.com` servers hold NS records pointing at your nameservers, so queries get sent to the right place.

**A common problem:** the NS records at your domain registrar do not match the NS records in your DNS zone. Then some queries go to the old provider and some to the new one, and users get different answers. This happens during a DNS provider migration.

```bash
dig NS example.com +short          # what the domain's zone says
dig NS example.com @a.gtld-servers.net    # what the .com registry says
```

If those two lists differ, that is the problem.

### What is a PTR record?

A **reverse** lookup. It maps an IP address back to a name. The opposite of an A record.

```
34.216.184.93.in-addr.arpa.   IN   PTR   example.com.
```

The IP address is written backwards and put under `in-addr.arpa`. That is just how reverse DNS is organised.

Where it matters:

- **Email.** Many mail servers refuse mail from an IP with no matching PTR record. This is the main reason PTR records exist in practice.
- **Logs.** Some tools turn IPs into names for readability.

**Note:** you cannot set your own PTR record. The owner of the IP address block controls it — your cloud provider or ISP. On AWS you request it through a support form for Elastic IPs.

```bash
dig -x 93.184.216.34        # reverse lookup
```

### What is DNS caching?

Saving DNS answers so you do not have to look them up again.

DNS is cached in many places, one after another:

```
Browser cache  →  OS cache  →  Local resolver  →  ISP / recursive resolver  →  Authoritative server
```

Each layer keeps the answer for as long as the TTL allows.

Why caching exists: it makes things much faster and it saves the authoritative servers from huge traffic. Without caching, every page load would need a full DNS lookup.

Why it causes trouble: **when you change a record, the old answer is still cached.** You cannot force other people's caches to clear. You have to wait for the TTL.

Clearing your own cache:

```bash
# Linux (systemd-resolved)
sudo systemd-resolve --flush-caches
# Linux (nscd)
sudo systemctl restart nscd
# macOS
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

**Also note:** some applications cache DNS themselves, separately from the OS. Java is famous for this — older JVM settings cache DNS forever. That causes traffic to keep going to a terminated server long after DNS was updated. The setting is `networkaddress.cache.ttl`.

### What is DNS TTL?

The number of seconds an answer may be cached before asking again.

```
www.example.com.   300   IN   A   93.184.216.34
                   ^^^ TTL in seconds
```

Choosing a value:

| TTL | Good for | Cost |
|---|---|---|
| 30–60 seconds | Records you may need to change quickly, failover | More DNS queries, more cost |
| 300 seconds (5 min) | A reasonable general default | Balanced |
| 3600 seconds (1 hour) | Records that rarely change | Changes take an hour to spread |
| 86400 seconds (1 day) | NS records, static things | Changes take a full day |

**The rule that matters:** lower the TTL **before** a planned change. Lower it to 60 seconds, wait for the old TTL to expire everywhere, then make the change, then raise it again afterwards.

If you lower the TTL at the same time as the change, it does not help at all. Caches already hold the old record with the old long TTL.

This also decides how fast DNS-based failover can be. If your TTL is 300 seconds, your failover takes at least 5 minutes.

### How do you troubleshoot DNS issues?

Work through the layers, from your own machine outward.

**1. What does your machine use for DNS?**

```bash
cat /etc/resolv.conf            # nameserver addresses and search domains
resolvectl status               # on systemd systems
```

**2. Does it resolve at all?**

```bash
dig example.com
nslookup example.com
```

Look at the `status` line in the `dig` output:

| Status | Meaning |
|---|---|
| `NOERROR` with an answer | Working |
| `NOERROR` with no answer | The name exists but has no record of that type |
| `NXDOMAIN` | The name does not exist |
| `SERVFAIL` | The server failed — often DNSSEC problems or the upstream is broken |
| `REFUSED` | The server refused to answer you |
| No response at all | Network or firewall blocking port 53 |

**3. Compare different DNS servers.** This finds most problems quickly.

```bash
dig @8.8.8.8 example.com            # public DNS
dig @1.1.1.1 example.com            # another public DNS
dig @<your-internal-dns> example.com
```

If the answers differ, you know exactly where the wrong or stale record lives.

**4. Check the authoritative server directly.** This skips all caches, so it shows the true current record.

```bash
dig NS example.com +short                       # find the nameservers
dig @ns1.example-dns.com example.com            # ask one of them directly
```

If the authoritative answer is correct but a resolver gives the old one, it is a caching problem and you must wait for the TTL.

**5. Trace the full path.**

```bash
dig +trace example.com
```

This shows every step from the root servers down. It reveals delegation problems.

**6. Check the network path to port 53.**

```bash
nc -zvu 8.8.8.8 53        # UDP
nc -zv  8.8.8.8 53        # TCP
```

DNS uses UDP normally and TCP for large answers. If UDP is blocked, small queries fail. If only TCP is blocked, small queries work and large ones fail — which produces very strange intermittent behaviour.

### What is split-horizon DNS?

The same domain name returns **different answers** depending on who is asking.

```
Query from inside the office   →  db.example.com  →  10.0.1.50   (private IP)
Query from the internet        →  db.example.com  →  203.0.113.5 (public IP)
```

Why it is used:

- Internal users go directly by private IP, which is faster and does not leave the network
- Internal names and addresses stay hidden from the public
- The same hostname works everywhere, so applications do not need different configuration per environment

On AWS this is done with a **Route 53 private hosted zone** for the same domain name as a public hosted zone. Queries from inside the VPC get the private zone; queries from outside get the public one.

**Why it confuses people:** two engineers get different answers to the same `dig`, and both are correct. If one person is on the VPN and one is not, they see a different world. Always check where a DNS query came from before deciding it is wrong.

---

## Load Balancing

### What is a load balancer?

A device or service that sits in front of several servers and spreads incoming requests across them.

It does four jobs:

1. **Distribute traffic** so no single server gets overloaded
2. **Check health** and stop sending traffic to broken servers
3. **Provide one stable address** so clients do not need to know about individual servers
4. **Allow scaling** — you can add or remove servers without changing anything for the client

Without a load balancer, one server failing means an outage for its users. With one, the traffic simply moves to the healthy servers.

### What are the types of load balancers?

**By layer:**

| Type | Works at | Decides based on |
|---|---|---|
| Layer 4 | Transport | IP address and port |
| Layer 7 | Application | URL path, hostname, headers, cookies |

**By where they run:**

| Type | Examples |
|---|---|
| Hardware | F5 BIG-IP, Citrix ADC |
| Software | NGINX, HAProxy, Envoy, Traefik |
| Cloud managed | AWS ALB / NLB, Google Cloud Load Balancing, Azure Load Balancer |
| DNS based | Route 53 weighted or latency routing |
| Global | AWS Global Accelerator, Cloudflare, Akamai |

**By where the traffic comes from:**

- **External (internet-facing)** — accepts traffic from the public internet
- **Internal** — only reachable inside your private network, used between services

### What is the difference between a Layer 4 and a Layer 7 load balancer?

| | Layer 4 | Layer 7 |
|---|---|---|
| Sees | IP and port only | The full HTTP request |
| Can route by path or hostname | No | Yes |
| Can terminate TLS | Sometimes (pass-through is normal) | Yes |
| Can modify headers | No | Yes |
| Can cache | No | Yes |
| Speed | Very fast, very low latency | Slightly slower |
| Protocols | Any TCP or UDP | HTTP, HTTPS, gRPC, WebSocket |
| AWS example | NLB | ALB |

**Layer 4** just forwards packets. It does not know or care what is inside. It is like a postal worker who moves sealed boxes without opening them.

**Layer 7** reads the request and can make decisions from the content:

```
example.com/api/*     →  the API servers
example.com/images/*  →  the image servers
admin.example.com     →  the admin servers
```

**When to use which:**

- Use **layer 7** for normal web traffic, because you want path routing, TLS termination, and header handling.
- Use **layer 4** for non-HTTP traffic (databases, message queues, game servers), when you need extreme performance, when you need a static IP address, or when the traffic must stay encrypted all the way to the server.

### What is a reverse proxy?

A server that sits in front of your application servers, receives requests from clients, and forwards them on.

The direction is the key to the name:

- A **forward proxy** sits in front of the **clients**. It acts on their behalf. Company web filters work this way.
- A **reverse proxy** sits in front of the **servers**. It acts on their behalf.

What a reverse proxy adds:

- TLS termination, so your app does not handle encryption
- Caching of responses
- Compression
- Rate limiting
- Request and response header changes
- Hiding your internal server layout
- Serving static files directly, without touching the app

**Reverse proxy or load balancer?** They overlap a lot. A load balancer's main job is spreading traffic across many servers. A reverse proxy's main job is being an intermediary, and it may only have one backend. NGINX does both at the same time, which is why the words are often used interchangeably.

### What is the difference between NGINX and HAProxy?

| | NGINX | HAProxy |
|---|---|---|
| Main purpose | Web server **and** reverse proxy | Proxy and load balancer only |
| Serves static files | Yes, very well | No |
| Layer 7 HTTP | Yes | Yes |
| Layer 4 TCP | Yes (`stream` module) | Yes, and this is its strength |
| Built-in stats page | Basic (paid version has more) | Yes, very detailed and free |
| Health checks | Basic in the free version | Advanced, free |
| Config style | Block based, nested | Flat sections |
| Also used as | Ingress controller, API gateway, cache | Ingress controller, database proxy |

**A short summary:** NGINX does more things, and can serve your website too. HAProxy does load balancing and does it extremely well, with better free health checks and statistics.

Both are excellent and either is a reasonable choice. In Kubernetes, `ingress-nginx` is the most common ingress controller, and **Envoy** is now the usual choice for service meshes and modern API gateways.

### What is a sticky session?

Sending the same client to the **same backend server** every time. Also called session affinity.

Why it is used: some applications keep the user's session in the memory of one server. If the next request goes to a different server, the user is logged out.

How it is done:

| Method | How it works |
|---|---|
| Cookie based | The load balancer sets a cookie naming the backend. Most reliable. |
| Source IP hash | Hash the client IP and always pick the same server from it. |
| Application cookie | The load balancer reads the app's own session cookie. |

**Why it is a problem, and worth avoiding:**

- **Uneven load.** One server can end up with far more users than the others.
- **Losing sessions.** If that server dies, its users lose their session anyway.
- **Blocks scaling.** New servers get no traffic until new users arrive.
- **Breaks deployments.** You cannot cleanly remove a server without dropping sessions.

**The better answer** is to make the application stateless. Keep sessions in Redis, in a database, or in a signed token (like a JWT) held by the client. Then any server can handle any request, and you do not need stickiness at all.

Sticky sessions are a workaround for an application design problem, not a feature to aim for.

### What is a health check?

An automatic test the load balancer runs against each backend to decide if it should receive traffic.

Typical settings:

| Setting | Meaning |
|---|---|
| Protocol and port | How to reach the check, for example HTTP on 8080 |
| Path | Which URL to request, for example `/health` |
| Interval | How often to check, for example every 10 seconds |
| Timeout | How long to wait for an answer, for example 5 seconds |
| Healthy threshold | How many passes in a row before adding it back |
| Unhealthy threshold | How many failures in a row before removing it |
| Expected code | Which status code counts as healthy, for example 200 |

**Shallow vs deep checks.** This is the important design decision.

A **shallow** check just returns 200. It only proves the process is alive.

A **deep** check tests the database and other dependencies too. It proves the app can actually do work.

Both have a risk. A shallow check can keep sending traffic to a server that cannot reach its database. But a deep check has a worse failure mode: **if the shared database goes slow, every server fails its health check at once, the load balancer removes all of them, and you have a total outage** — caused by your own health check.

The usual compromise is a shallow check for the load balancer, and separate monitoring and alerting on the dependencies.

### What is the difference between Round Robin and Least Connections?

| | Round Robin | Least Connections |
|---|---|---|
| How it picks | The next server in order, one after another | The server with the fewest open connections |
| Assumes | All requests cost about the same | Requests vary in cost |
| Best for | Short, similar requests | Long or uneven requests |
| Cost to compute | Almost none | Slightly more, must track counts |

**Round robin** just goes 1, 2, 3, 1, 2, 3. Simple and fair when every request is similar.

It goes wrong when requests are very different. Imagine server 1 gets a 10-second file upload and server 2 gets a 5ms health check. Round robin keeps sending equally, so server 1 gets overloaded while server 2 sits idle.

**Least connections** handles that. It sends the next request to whichever server is least busy right now.

Other algorithms worth knowing:

| Algorithm | Idea |
|---|---|
| Weighted round robin | Bigger servers get more requests |
| IP hash | The same client IP always goes to the same server (a form of stickiness) |
| Least response time | Combines connection count with how fast the server is answering |
| Random with two choices | Pick two at random, then take the less busy one. Almost as good as least connections and much cheaper. |

Practical note: AWS ALB uses round robin by default and supports least outstanding requests. NLB uses a flow hash, which keeps one connection on one target.

### What is connection draining?

Letting existing connections finish before you remove a server from the load balancer.

Without draining, the load balancer removes the server immediately and every in-progress request is cut off. Users see errors, even though you were doing a normal deploy.

With draining, the sequence is:

```
1. Stop sending NEW requests to the server
2. Let the current requests finish       ← this is the draining time
3. Only then remove the server
```

The setting has different names:

| Platform | Name |
|---|---|
| AWS ALB / NLB | Deregistration delay (default 300 seconds) |
| Kubernetes | `terminationGracePeriodSeconds` and the `preStop` hook |
| NGINX | `worker_shutdown_timeout` |
| HAProxy | Set the server to `drain` state |

**How to choose the value:** slightly longer than your longest normal request. If most requests take under 5 seconds, 30 seconds is plenty. The AWS default of 300 seconds is often far too long — it makes every deploy slow.

**One extra detail that catches people out.** Removing the server from the load balancer and telling the application to stop are two separate events, and they can happen at nearly the same time. So a request may arrive just after the app started shutting down.

The usual fix is a short sleep before shutdown begins — in Kubernetes, a `preStop` hook that sleeps 5 seconds. That gives the load balancer time to notice the server is gone before the app stops accepting connections. Without it, you see a few errors on every single deploy.

---

## Linux Networking

### How do you check IP addresses?

```bash
ip addr                    # all interfaces and their addresses
ip addr show eth0          # one interface
ip -4 addr                 # IPv4 only
ip -br addr                # short, one line per interface
hostname -I                # just the addresses
```

Reading the output:

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 10.0.1.25/24 brd 10.0.1.255 scope global eth0
```

- `eth0` — the interface name
- `UP` — the interface is enabled; `LOWER_UP` means the cable or link is actually connected
- `mtu 1500` — the maximum packet size
- `inet 10.0.1.25/24` — the IPv4 address and its subnet

To find your **public** IP (which is different if you are behind NAT):

```bash
curl ifconfig.me
```

`ifconfig` is the old command. It still works on many systems, but `ip` is the current one and shows more.

### What is the difference between ping and traceroute?

| | ping | traceroute |
|---|---|---|
| Answers | "Can I reach it, and how fast?" | "What path does traffic take?" |
| Shows | One round trip time | Every router along the way |
| Uses | ICMP Echo | Increasing TTL values |
| Speed | Instant | Slower, tests each hop |

**ping** sends a packet and measures how long the reply takes.

```bash
ping -c 4 example.com
```

**traceroute** finds each router on the path. It works by sending packets with TTL 1, then 2, then 3. Each router that drops a packet for expired TTL sends back an ICMP message revealing its address.

```bash
traceroute example.com
traceroute -T -p 443 example.com     # use TCP instead, better when ICMP is blocked
```

**Important limits of both:**

- Many firewalls block ICMP. So `ping` failing does not mean the host is down, and traceroute often shows `* * *` for hops that do not reply. That is usually normal, not a fault.
- Routers give low priority to replying to ICMP. So a high time on one hop does not always mean that hop is slow. Look at the **final** hop time and the overall trend, not one middle hop.

**`mtr` is better than both.** It runs traceroute continuously and shows loss per hop, so you can see where packets are actually being dropped:

```bash
mtr example.com
mtr --report --report-cycles 100 example.com
```

### What does netstat do?

Shows network connections, listening ports, routing tables, and interface statistics.

```bash
netstat -tulnp        # listening ports with the process name
netstat -an           # all connections
netstat -s            # protocol statistics, useful for retransmissions
netstat -rn           # routing table
```

The flags mean: `t` TCP, `u` UDP, `l` listening only, `n` numbers not names, `p` show the process.

**It is deprecated.** `netstat` reads `/proc`, which is slow on a busy server with many connections. Use `ss` instead. `netstat` may not even be installed on newer systems (it is in the `net-tools` package).

It is still worth knowing because you will see it in old documentation and scripts, and `netstat -s` is still handy for protocol counters.

### What does ss do?

The same job as `netstat`, but faster and with more filtering. `ss` means socket statistics.

```bash
ss -tulnp                          # listening TCP and UDP ports with process names
ss -tan                            # all TCP connections
ss -tn state established           # only established connections
ss -tn state time-wait | wc -l     # count TIME_WAIT connections
ss -ti                             # detailed info per connection: RTT, retransmits, window
ss -tn dst 10.0.1.5                # only connections to one address
ss -tn '( dport = :443 or sport = :443 )'    # filter by port
```

Why it is faster: `ss` asks the kernel directly through netlink, instead of reading and parsing files in `/proc`. On a server with 100,000 connections this is the difference between instant and many seconds.

The most useful one for debugging performance is `ss -ti`. It shows the round trip time and retransmission count **per connection**, which tells you if the network is the problem.

### How do you check listening ports?

```bash
ss -tulnp              # the standard way
netstat -tulnp         # older
lsof -i -P -n          # by process, needs root for other users' processes
lsof -i :443           # who is using port 443
fuser 443/tcp          # which process ID owns port 443
```

Reading the output, one thing matters most — the address it is bound to:

```
LISTEN  0  128   0.0.0.0:443     ← all interfaces, reachable from outside
LISTEN  0  128   127.0.0.1:8080  ← localhost ONLY, not reachable from outside
LISTEN  0  128   10.0.1.25:9090  ← only on that one interface
```

**This is a very common cause of "the port is open but I cannot connect".** The app is bound to `127.0.0.1` instead of `0.0.0.0`. It works when you test from the machine itself and fails from everywhere else. Check the bind address before checking the firewall.

### What is tcpdump?

A tool that captures the actual network packets going in and out of an interface, so you can see exactly what is on the wire.

You use it when logs are not enough — when you need to prove whether a packet arrived at all, or see exactly what was sent.

```bash
tcpdump -i eth0                                   # everything on eth0 (very noisy)
tcpdump -i eth0 port 443                          # only port 443
tcpdump -i eth0 host 10.0.1.50                    # only traffic to or from one host
tcpdump -i eth0 'tcp port 80 and host 10.0.1.50'  # combine filters
tcpdump -i eth0 -n port 5432                      # -n: do not resolve names (faster)
tcpdump -i eth0 -c 100                            # stop after 100 packets
tcpdump -i eth0 -A port 80                        # print packet contents as text
tcpdump -i eth0 -w capture.pcap                   # save to a file for Wireshark
tcpdump -i any port 53                            # all interfaces
```

Useful filters for real problems:

```bash
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'    # only SYN packets (connection attempts)
tcpdump -i eth0 'tcp[tcpflags] & tcp-rst != 0'    # only RST (resets — connections being cut)
tcpdump -i eth0 icmp                              # ICMP, useful for MTU problems
```

**Always use a filter.** On a busy server, capturing everything creates enormous output and adds load.

### How do you capture packets?

The general method:

**1. Capture on the right interface.** Use `ip addr` to find it. Use `-i any` if you are not sure.

**2. Filter as tightly as you can**, by port and host, to keep the output small.

**3. Save to a file** if you want to analyse it properly:

```bash
tcpdump -i eth0 -w /tmp/capture.pcap port 443 -c 5000
```

**4. Read it, or open it in Wireshark:**

```bash
tcpdump -r /tmp/capture.pcap                # read back on the server
tcpdump -r /tmp/capture.pcap -A | less      # with contents as text
```

Wireshark gives you a graphical view and can follow a whole TCP conversation, which is much easier to read.

**What to look for:**

| What you see | What it means |
|---|---|
| SYN sent, no reply at all | Firewall dropping the packet silently |
| SYN sent, RST comes back | Nothing is listening on that port |
| Many retransmissions of the same packet | Packet loss |
| Data flowing, then a sudden RST | Something cut the connection: a timeout, or a proxy |
| Big packets missing, small ones fine | MTU problem |

**Two practical notes:**

- Capturing on a busy interface uses CPU. Use filters and `-c` to limit it.
- HTTPS traffic is encrypted, so you can see the connection and its timing but not the content. To see the content, capture on the machine **before** encryption, or terminate TLS at a proxy you control.

### What is `ip route`?

The command that shows and changes the routing table. The routing table decides where a packet goes for a given destination.

```bash
ip route                          # show the routing table
ip route show                     # same thing
ip route get 8.8.8.8              # which route WOULD be used for this address
```

Reading the output:

```
default via 10.0.1.1 dev eth0                      ← anything not matched goes here
10.0.1.0/24 dev eth0 proto kernel scope link       ← my own subnet, reachable directly
172.17.0.0/16 dev docker0                          ← Docker's network
```

How Linux decides: it looks for the **most specific match** (the longest prefix). `10.0.1.0/24` beats `default` for an address inside that range. If nothing matches, the default route is used.

`ip route get` is the most useful of these when debugging. It tells you exactly which route and interface a packet will use, without guessing:

```bash
ip route get 10.0.5.20
# 10.0.5.20 via 10.0.1.1 dev eth0 src 10.0.1.25
```

Adding routes (temporary, lost on reboot):

```bash
sudo ip route add 10.0.5.0/24 via 10.0.1.1
sudo ip route del 10.0.5.0/24
```

### How do you check routing tables?

```bash
ip route                    # main table
ip route show table all     # all tables, including policy routing
ip rule show                # rules that choose which table to use
ip route get <ip>           # test one destination
netstat -rn                 # older command
route -n                    # oldest command
```

A system can have several routing tables. `ip rule` decides which one to use, based on things like the source address. This is used for VPNs, containers, and multiple network interfaces. If `ip route` looks correct but traffic still goes the wrong way, check `ip rule`.

For debugging, the most useful sequence is:

```bash
ip addr                        # what address do I have?
ip route get <destination>     # which route will be used?
ping <the gateway>             # can I reach my gateway?
mtr <destination>              # where does the path break?
```

### What is iptables?

The traditional Linux firewall. It filters packets based on rules.

The structure:

**Tables** hold **chains**, and chains hold **rules**.

Main tables:

| Table | Purpose |
|---|---|
| `filter` | Allow or block packets. The default. |
| `nat` | Change addresses (NAT, port forwarding) |
| `mangle` | Change packet fields |
| `raw` | Skip connection tracking |

Main chains in the filter table:

| Chain | When it runs |
|---|---|
| `INPUT` | Packets arriving for this machine |
| `OUTPUT` | Packets leaving from this machine |
| `FORWARD` | Packets passing through this machine to somewhere else |

```bash
iptables -L -n -v                     # list rules with counters
iptables -t nat -L -n                 # list NAT rules
iptables -L INPUT -n --line-numbers   # numbered, so you can delete by number
iptables -S                           # show as commands, easy to save
```

Rules run **in order, top to bottom**, and the **first match wins**. So rule order matters a lot. A broad ACCEPT rule near the top makes all the rules below it useless.

The packet counters in `-v` output are very useful for debugging. If a rule has 0 packets, nothing is matching it.

**Note for Kubernetes:** `kube-proxy` writes large numbers of iptables rules to make Services work. So on a Kubernetes node, `iptables -L` output is huge and mostly generated. Do not edit it by hand.

### What is nftables?

The modern replacement for iptables. It does the same job with a cleaner design.

Improvements over iptables:

- One tool instead of four (`iptables`, `ip6tables`, `arptables`, `ebtables`)
- One combined ruleset for IPv4 and IPv6
- Faster with large rulesets, because it uses sets and maps instead of long lists
- Rules can be changed atomically, so there is no moment where the firewall is half-updated

```bash
nft list ruleset                          # show everything
nft list tables                           # show tables
nft add rule inet filter input tcp dport 443 accept
```

Most modern Linux systems use nftables underneath, even when you type `iptables` commands — there is a compatibility layer (`iptables-nft`) that translates them. So both commands may show different views of the same rules, which is confusing. Check which one is really in use:

```bash
iptables --version
# iptables v1.8.7 (nf_tables)     ← nftables underneath
# iptables v1.8.7 (legacy)        ← real iptables
```

### How do you check firewall rules?

It depends on the layer. Check all of them, because any one can block traffic.

**On the Linux machine:**

```bash
iptables -L -n -v                # iptables
nft list ruleset                 # nftables
ufw status verbose               # Ubuntu's simple firewall
firewall-cmd --list-all          # RHEL / CentOS firewalld
```

**On AWS, outside the machine:**

- **Security group** — attached to the instance or ENI. Stateful.
- **Network ACL** — attached to the subnet. Stateless, so check both directions.
- **Route tables** — a missing route looks exactly like a firewall block.

**Also check:**

- SELinux or AppArmor, which can block network access even when the firewall allows it (`getenforce`, `ausearch -m avc`)
- Whether the application is bound to `127.0.0.1` instead of `0.0.0.0`

**The quick way to tell the difference between a firewall block and nothing listening:**

```bash
nc -zv host 443
```

| Result | Meaning |
|---|---|
| Connection **refused**, immediately | You reached the machine. Nothing is listening on that port. |
| Connection **timed out**, after a wait | A firewall silently dropped the packet. You did not reach the machine. |

That single distinction saves a lot of time. Refused means look at the application. Timeout means look at the firewall.

### What is curl used for?

Sending network requests from the command line. It is the main tool for testing HTTP APIs and web servers.

```bash
curl https://example.com                       # GET, print the body
curl -I https://example.com                    # headers only (HEAD request)
curl -v https://example.com                    # verbose: show the whole exchange
curl -L https://example.com                    # follow redirects
curl -o file.html https://example.com          # save to a file
curl -s https://example.com                    # silent, no progress bar
curl -k https://example.com                    # ignore certificate errors (testing only)
curl --resolve example.com:443:1.2.3.4 https://example.com    # test one specific server
curl -X POST -d '{"a":1}' -H 'Content-Type: application/json' https://api.example.com
curl -u user:pass https://example.com          # basic authentication
curl --max-time 10 https://example.com         # give up after 10 seconds
```

**The most useful flag for SRE work** is the timing breakdown. It shows exactly where the time goes:

```bash
curl -w "dns: %{time_namelookup}s  connect: %{time_connect}s  tls: %{time_appconnect}s  ttfb: %{time_starttransfer}s  total: %{time_total}s\n" \
     -o /dev/null -s https://example.com
```

Example output:

```
dns: 0.004s  connect: 0.021s  tls: 0.089s  ttfb: 0.412s  total: 0.415s
```

Now you know the answer immediately. DNS is fast, connecting is fast, TLS is fine, but the server took 0.4 seconds to start answering. So it is a server-side problem, not a network problem. This one command replaces a lot of guessing.

The `--resolve` flag is also very useful. It lets you send a request to one specific server IP while still using the correct hostname for TLS and the `Host` header. That is how you test a single backend behind a load balancer.

### What is the difference between curl and wget?

| | curl | wget |
|---|---|---|
| Main purpose | Send requests and see the response | Download files |
| Default output | Prints to the screen | Saves to a file |
| Follows redirects | Only with `-L` | Yes, by default |
| Recursive download | No | Yes, can mirror a whole site |
| Continue a broken download | Limited | Yes, `-c` |
| Protocols | Very many (HTTP, FTP, SFTP, SMTP, LDAP and more) | HTTP, HTTPS, FTP |
| Works as a library | Yes, libcurl is used everywhere | No |
| Best for | Testing APIs and debugging | Downloading and mirroring |

Short version: **use curl to test and inspect, use wget to download.**

For SRE work, curl is used far more often, because you usually want to see headers, status codes, and timing rather than save the file.

---

## Kubernetes Networking

### How do Pods communicate?

Every Pod gets its own IP address, and **any Pod can reach any other Pod directly by IP, with no NAT.** This is a rule that Kubernetes requires.

Three cases:

**1. Containers in the same Pod.** They share one network namespace, so they share the same IP and port space. They talk over `localhost`.

```
container A → localhost:8080 → container B
```

Because they share ports, two containers in one Pod cannot both listen on port 8080.

**2. Pod to Pod on the same node.** Traffic goes through a virtual bridge on the node. It never leaves the machine.

**3. Pod to Pod on different nodes.** The CNI plugin handles this, either by real routing or by wrapping the packet (see the cross-node question below).

**In practice, you do not use Pod IPs.** Pod IPs change whenever a Pod restarts. You use a Service name instead, which stays the same:

```
http://payments.default.svc.cluster.local:8080
http://payments:8080          (short form, same namespace)
```

### What is CNI?

CNI means Container Network Interface. It is a plugin standard. Kubernetes does not implement Pod networking itself — a CNI plugin does.

Any CNI plugin must provide:

1. Every Pod gets its own IP address
2. Every Pod can reach every other Pod without NAT
3. Nodes can reach Pods, and Pods can reach nodes

How each plugin does it is up to them:

| Plugin | How it works | Notes |
|---|---|---|
| **AWS VPC CNI** | Gives Pods real VPC IP addresses | No wrapping, fast. But uses up subnet IPs quickly. |
| **Calico** | BGP routing, or VXLAN wrapping | Strong network policy support |
| **Cilium** | eBPF in the kernel | Very fast, can replace kube-proxy, layer 7 policies |
| **Flannel** | VXLAN wrapping | Simple, but no network policy |

**The AWS VPC CNI detail that matters most:** because Pods get real VPC IPs, you can run out of IP addresses in your subnet. Each instance type also has a limit on how many IPs it can hold. When you hit either limit, new Pods stay `Pending` with no obvious resource shortage. The fixes are bigger subnets, or prefix delegation (`ENABLE_PREFIX_DELEGATION=true`), which assigns blocks of IPs instead of single ones.

### What is kube-proxy?

A component on every node that makes Services work. It watches Services and their endpoints, then writes rules so that traffic sent to a Service IP is redirected to a real Pod IP.

The important detail: **in the normal iptables mode, kube-proxy is not in the traffic path at all.** It only writes rules. The Linux kernel does the actual forwarding.

That is why Service load balancing is fast, and also why it is quite simple — it picks a Pod roughly at random per connection. It cannot see how busy a Pod is, and it cannot balance individual HTTP requests inside one connection.

Some CNI plugins, such as Cilium, replace kube-proxy entirely with eBPF.

### What are the iptables and IPVS modes?

Two ways kube-proxy can implement Services.

| | iptables mode | IPVS mode |
|---|---|---|
| Built on | iptables rules | The kernel's IPVS load balancer |
| Rule lookup | Walks a list, so it slows down with more Services | Hash table, so it stays fast |
| Performance at scale | Degrades past a few thousand Services | Stays constant |
| Load balancing options | Random only | Round robin, least connection, source hash, and more |
| Default | Yes, in most clusters | No, must be enabled |

**Why IPVS exists:** iptables rules are checked in order. With 5,000 Services, that is a very long list to walk for every connection, and adding a Service means rewriting a large ruleset. IPVS uses a hash table, so lookup time does not grow.

For most clusters, iptables mode is completely fine. IPVS matters for very large clusters.

There is also a newer `nftables` mode, and eBPF-based replacements which avoid the problem entirely.

### How does a Service work?

A Service gives a stable name and IP for a group of Pods that keep changing.

The steps:

```
1. You create a Service with a label selector, for example app=api
2. The endpoints controller watches for Pods that match that label AND are Ready
3. It keeps a list of their IPs in an EndpointSlice object
4. kube-proxy on every node reads that list and writes forwarding rules
5. CoreDNS creates a DNS record for the Service name
6. A client looks up the name, gets the Service IP, and sends traffic there
7. The kernel rewrites the destination to one of the real Pod IPs
```

Two things decide whether traffic reaches a Pod:

- **The label selector must match the Pod's labels** — exactly
- **The Pod must be Ready** — a Pod failing its readiness probe is removed from the list

That second point is how rolling updates avoid dropping traffic.

### What is the difference between ClusterIP, NodePort and LoadBalancer?

They build on each other. NodePort includes ClusterIP behaviour. LoadBalancer includes both.

| Type | Reachable from | How it works |
|---|---|---|
| **ClusterIP** | Inside the cluster only | A virtual IP that only exists inside the cluster |
| **NodePort** | Any node's IP, on a high port | Opens the same port (30000–32767) on **every** node |
| **LoadBalancer** | The internet, or your private network | Asks the cloud provider to create a real load balancer |

```
ClusterIP:     10.96.0.10:80              (internal only)
NodePort:      <any-node-ip>:31234        (also has a ClusterIP)
LoadBalancer:  <cloud-lb-address>:80      (also has a NodePort and a ClusterIP)
```

Two more types:

- **ExternalName** — no proxying at all. It just returns a CNAME to an external DNS name. Useful for giving an RDS endpoint a cluster-internal name.
- **Headless** (`clusterIP: None`) — no virtual IP. DNS returns all the individual Pod IPs. Used by StatefulSets and by clients that do their own load balancing, such as gRPC and Kafka.

**The cost point:** each `LoadBalancer` Service creates one real cloud load balancer, which you pay for. Twenty Services means twenty load balancers. That is the main reason to use an Ingress instead.

### What is an Ingress?

A set of HTTP routing rules. One entry point that sends traffic to different Services based on hostname and URL path, with TLS.

```
api.example.com/users     →  users-service
api.example.com/orders    →  orders-service
admin.example.com/*       →  admin-service
```

An Ingress object is **only configuration**. On its own it does nothing. It needs an Ingress Controller running in the cluster to read it and do the actual work.

Ingress handles HTTP and HTTPS only. For raw TCP or UDP, such as a database or a game server, you still need a `LoadBalancer` Service.

The newer **Gateway API** is designed to replace Ingress. It is more flexible and handles more protocols.

### What is an Ingress Controller?

The component that reads Ingress objects and configures a real proxy or cloud load balancer to match.

Common ones:

| Controller | What it does |
|---|---|
| **ingress-nginx** | Runs NGINX inside the cluster, usually behind one LoadBalancer Service |
| **AWS Load Balancer Controller** | Creates a real ALB for your Ingress |
| **Traefik** | Runs in the cluster, popular for its simple configuration |
| **Istio Gateway** | Part of the Istio service mesh |

**If no controller is installed, your Ingress does nothing.** It will have no address, and no traffic will flow. This is a very common cause of "my Ingress is not working" in a new cluster.

If you have several controllers, `ingressClassName` decides which one handles which Ingress. If it is missing or wrong, no controller claims the Ingress and it is silently ignored.

### How does CoreDNS work?

CoreDNS is the DNS server inside the cluster. It runs as a normal Deployment in the `kube-system` namespace.

How Pods use it:

1. When a Pod starts, the kubelet writes `/etc/resolv.conf` inside it, pointing at the `kube-dns` Service IP
2. The Pod's application does a normal DNS lookup
3. CoreDNS answers, either from cluster records or by forwarding to an upstream DNS server

Inside a Pod, `/etc/resolv.conf` looks like this:

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

The `search` line is why short names work. When you ask for `payments`, the resolver tries `payments.default.svc.cluster.local` first, then the other suffixes.

Service names follow this pattern:

```
<service>.<namespace>.svc.cluster.local

payments                                 (same namespace)
payments.prod                            (different namespace)
payments.prod.svc.cluster.local          (full name)
```

**Two operational points:**

- **CoreDNS is a shared dependency.** If it is slow or unhealthy, everything in the cluster looks broken in confusing ways. Give it proper resource requests and monitor its latency and error rate.
- **`ndots:5` causes extra lookups.** For an external name like `api.stripe.com`, which has fewer than 5 dots, the resolver tries all the cluster suffixes first and gets NXDOMAIN each time, before finally trying the real name. That is 4 wasted queries per lookup. For high-traffic services, lowering `ndots` or adding a trailing dot (`api.stripe.com.`) reduces DNS load a lot.

### What are Network Policies?

Firewall rules for Pods, written as Kubernetes objects. They select Pods by label and define what traffic is allowed in and out.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - port: 5432
```

That means: Pods labelled `app: database` accept traffic on port 5432, but only from Pods labelled `app: api`.

**Three things that catch people out:**

1. **Pod networking allows everything by default.** Network policies only take effect once a policy selects a Pod. Then that Pod becomes deny-by-default for the directions you listed.

2. **Your CNI must support them.** This is the important one. If your plugin does not enforce network policies, Kubernetes still accepts the object and shows it as created — and simply ignores it. On EKS with the VPC CNI you must enable network policy support, or run Calico. Otherwise you have a false sense of security.

3. **Do not forget DNS.** If you add a deny-all egress policy and forget to allow UDP port 53 to CoreDNS, every Pod loses DNS and everything breaks. This is the single most common network policy mistake.

To verify a policy actually works, test it. Do not assume:

```bash
kubectl run test --rm -it --image=nicolaka/netshoot -- curl -m 5 http://database:5432
```

### Why does every Pod have its own IP?

Because it removes a large amount of complexity.

The alternative is port mapping, which is what plain Docker does — many containers share the host IP, each on a different port. That creates problems:

- **Port conflicts.** Two Pods both want port 8080. Someone has to allocate ports.
- **Applications must be told their port**, which they often cannot handle.
- **Service discovery gets harder**, because you must track both IP and port.
- **Network policy gets harder**, because you cannot identify a workload by IP.

With one IP per Pod:

- Every Pod can use its natural port. A web server just listens on 8080.
- Pods look like normal machines to the network, so existing tools work.
- Network policy can identify Pods by IP.
- No NAT between Pods, so the source IP is preserved and logs make sense.

The cost is that you need a lot of IP addresses. That is exactly why the AWS VPC CNI can run out of subnet IPs.

### How does cross-node Pod networking work?

A Pod on node A sends traffic to a Pod on node B using its Pod IP directly. No NAT. How that is delivered depends on the CNI:

**1. Native routing (AWS VPC CNI).** Pod IPs are real VPC IP addresses. The VPC route table already knows how to deliver them. No wrapping, so almost no overhead.

**2. Overlay / encapsulation (VXLAN, Geneve — Flannel, Calico in VXLAN mode).** The original packet is wrapped inside another packet addressed node-to-node. Node B unwraps it and delivers it. This works on any network, but adds overhead and reduces the usable MTU.

**3. BGP (Calico native).** Each node announces which Pod IPs it owns to the other nodes or to the physical network. Then normal routing delivers the traffic.

**When cross-node traffic breaks, check these three things:**

| Cause | How to spot it |
|---|---|
| Security group or firewall between nodes | Same-node traffic works, cross-node does not |
| **MTU mismatch** with an overlay | Small requests work, large ones hang. Very distinctive. |
| CNI pod crashed on one node | Only Pods on that one node are affected |

The MTU one is worth remembering. Wrapping adds bytes, so the usable MTU inside the overlay is smaller than 1500. If the MTU is set wrong, small packets fit and large ones are dropped — so a health check passes and a real request hangs.

### What happens when a Service has no endpoints?

Traffic to the Service fails. Usually you get "connection refused" immediately, because there is nothing to forward to.

Check the endpoints:

```bash
kubectl get endpointslices -l kubernetes.io/service-name=<service>
kubectl describe svc <service>
```

The causes, in order of how often they happen:

**1. The label selector does not match the Pod labels.** The most common cause by far. One character difference is enough.

```bash
kubectl get svc <service> -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels
```

**2. No Pods are Ready.** The Pods exist and are running, but the readiness probe is failing, so they are excluded on purpose. Check the READY column:

```bash
kubectl get pods -l app=<label>
# NAME         READY   STATUS
# api-abc123   0/1     Running     ← running, but NOT ready
```

**3. Wrong port.** The Service's `targetPort` does not match the port the container actually listens on. Or a named port does not exist in the Pod spec.

**4. Wrong namespace.** A Service can only select Pods in its own namespace.

**5. No selector at all.** Valid for `ExternalName` Services, or if you manage the endpoints manually — but then you have to maintain them yourself.

### How do you troubleshoot DNS failures in Kubernetes?

**1. Is CoreDNS healthy?**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

If CoreDNS is crashing or not Ready, that explains everything.

**2. Does the kube-dns Service have endpoints?**

```bash
kubectl get svc -n kube-system kube-dns
kubectl get endpointslices -n kube-system -l kubernetes.io/service-name=kube-dns
```

**3. Test from inside a Pod.**

```bash
kubectl run dnstest --rm -it --image=nicolaka/netshoot -- bash

# then inside:
cat /etc/resolv.conf              # is the nameserver correct?
nslookup kubernetes.default       # internal name
nslookup google.com               # external name
dig @10.96.0.10 payments.default.svc.cluster.local
```

**4. Read the result.** The pattern tells you where the problem is:

| Result | Meaning |
|---|---|
| Internal fails, external fails | CoreDNS is down, or the Pod cannot reach it at all |
| Internal works, external fails | CoreDNS forwarding to upstream is broken, or egress is blocked |
| Internal fails, external works | The record does not exist, or the search domains are wrong |
| Everything works but is slow | CoreDNS is overloaded, or the conntrack race (see below) |

**5. Check for a NetworkPolicy blocking DNS.** Very common after adding deny-all. DNS needs UDP port 53 to the `kube-system` namespace.

```bash
kubectl get networkpolicy -A
```

**6. Check the `dnsPolicy` on the Pod.** If it is set to `Default` instead of `ClusterFirst`, the Pod uses the node's DNS and cannot resolve cluster names at all.

**7. If lookups take about 5 seconds, then succeed** — that is the well-known UDP conntrack race condition, not a broken resolver. The fixes are NodeLocal DNSCache, lowering `ndots`, or using TCP for DNS.

### How do you troubleshoot Pod-to-Pod communication failures?

First, work out the shape of the problem. That removes most causes immediately.

Start a debug Pod:

```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
```

Then run these tests in order:

```bash
ping <pod-ip>                  # 1. raw IP reachability
curl <pod-ip>:<port>           # 2. is the app listening?
nslookup <service>.<ns>        # 3. does DNS work?
curl <service>.<ns>:<port>     # 4. does the Service work?
```

Now read the pattern:

| What happens | Where the problem is |
|---|---|
| Pod IP works, Service name does not | DNS or the Service/endpoints, **not** the network |
| Service resolves but connection refused | The app is not listening on that port, or `targetPort` is wrong |
| Same-node works, cross-node fails | CNI, node security group, or route table |
| Small requests work, large ones hang | **MTU mismatch** |
| Nothing works between certain namespaces | A NetworkPolicy is blocking it |
| Works sometimes, fails sometimes | Some Pods are unhealthy but still in the endpoint list |

Other things to check:

```bash
kubectl get pod <pod> -o wide           # does the Pod even have an IP?
kubectl get pods -n kube-system -o wide | grep -i cni    # is the CNI running on both nodes?
kubectl get netpol -A                   # any network policies?
```

**If a Pod has no IP address at all**, the CNI failed to assign one. On the AWS VPC CNI that usually means the subnet is out of IP addresses, or the instance has reached its ENI/IP limit.

---

## AWS Networking

### What is a VPC?

VPC means Virtual Private Cloud. It is your own private network inside AWS.

You choose an IP range (a CIDR block) when you create it, for example `10.0.0.0/16`. Everything you build — instances, databases, load balancers — goes inside it.

A VPC:

- Belongs to one **region**
- Spans all the **availability zones** in that region
- Is completely isolated from other VPCs unless you connect them on purpose

Choose the CIDR carefully, because it is hard to change later. `10.0.0.0/16` gives 65,536 addresses, which is a good default. You can add extra CIDR blocks later, but you cannot shrink or replace the first one.

### What are public and private subnets?

A subnet is a slice of your VPC, and it lives in **one** availability zone.

The difference between public and private is **only the route table**. There is no setting called "public".

| | Public subnet | Private subnet |
|---|---|---|
| Route to `0.0.0.0/0` | Points to an **Internet Gateway** | Points to a **NAT Gateway**, or nothing |
| Can receive traffic from the internet | Yes, if resources have a public IP | No |
| Can reach the internet outbound | Yes | Yes, through the NAT Gateway |

A typical layout:

```
Public subnets   →  load balancers, NAT gateways, bastion hosts
Private subnets  →  application servers, containers
Private subnets  →  databases (often with no internet route at all)
```

The rule: anything that does not need to be reached from the internet goes in a private subnet. That is most things.

Remember that a subnet is tied to one AZ. So for a highly available system you need at least one subnet per AZ, usually three.

### What is an Internet Gateway?

A component you attach to your VPC to allow traffic between the VPC and the internet.

It does two things:

1. Provides a route for internet traffic
2. Performs NAT for instances that have a public IP address

For an instance to actually reach the internet, **three** things must all be true:

1. There is an Internet Gateway attached to the VPC
2. The subnet's route table has a route `0.0.0.0/0` → the Internet Gateway
3. The instance has a **public IP** or an **Elastic IP**

If any one is missing, it does not work. The third one is the most commonly forgotten.

An Internet Gateway is free, highly available, and scales automatically. There is nothing to size or manage.

### What is a NAT Gateway?

A managed service that lets instances in **private** subnets reach the internet, while stopping the internet from reaching them.

How it works: the private instance sends traffic to the NAT Gateway. The NAT Gateway replaces the source IP with its own public IP and forwards the traffic. Replies come back to the NAT Gateway, which sends them to the original instance.

Important facts:

- It lives in a **public** subnet, and needs an Elastic IP
- It is **one-way**. Outbound only. Nothing on the internet can start a connection through it.
- It is **per availability zone**. One NAT Gateway serves one AZ, so for high availability you need one in each AZ. If you use a single NAT Gateway for all AZs, losing that AZ breaks internet access for everything.
- It **costs money** — an hourly charge plus a charge per gigabyte processed. This is a common surprise on AWS bills.

**Two ways to reduce the cost:**

1. Use **VPC endpoints** for AWS services like S3 and ECR. Then that traffic never goes through the NAT Gateway.
2. Do not send traffic between private subnets through the NAT Gateway. Only internet traffic should go there.

There is also a **NAT instance**, which is an EC2 instance doing the same job. It is cheaper for very small workloads but you have to manage and scale it yourself. The NAT Gateway is the standard choice.

### What is a Route Table?

A list of rules that decides where network traffic goes, based on the destination address.

Each subnet is associated with exactly one route table.

Example:

```
Destination        Target
10.0.0.0/16        local              ← inside the VPC, handled automatically
0.0.0.0/0          igw-12345          ← everything else goes to the internet gateway
```

The `local` route is created automatically and cannot be removed. It covers the whole VPC CIDR, which is why anything inside your VPC can reach anything else inside it by default.

**How AWS chooses:** the most specific route wins (longest prefix match). So `10.0.5.0/24` beats `10.0.0.0/16`, and both beat `0.0.0.0/0`.

Route tables are how you make a subnet public or private, connect VPCs, or send traffic through a firewall appliance.

**A tip for debugging:** a missing route looks exactly like a firewall block — the connection just times out. So when traffic does not flow, check the route table before spending time on security groups.

### What is a Security Group?

A firewall that is attached to a resource — an EC2 instance, an ENI, an RDS database, a load balancer.

Key properties:

- **Stateful.** If you allow traffic in, the reply is automatically allowed out. You do not need a matching outbound rule.
- **Allow rules only.** You cannot write a deny rule. Anything not allowed is blocked.
- **Default outbound is allow all.** Default inbound is deny all.
- You can reference **another security group** as the source, instead of an IP range. This is very useful and very common:

```
Allow inbound on 5432 from sg-app-servers
```

That means "any instance with the app-servers security group may connect to the database". You do not need to know or maintain their IP addresses. When you add a new app server, it works automatically.

- A resource can have several security groups. The rules are combined — if any group allows the traffic, it is allowed.

### What is a Network ACL?

A firewall attached to a **subnet**, not to a resource. Every resource in the subnet is affected.

Key properties:

- **Stateless.** This is the big difference. You must allow traffic in **and** allow the reply out. If you allow inbound port 443 but do not allow outbound to the ephemeral port range, the reply is blocked and the connection fails.
- Supports **both allow and deny** rules.
- Rules are **numbered**, and processed in order from lowest to highest. The first match wins, and processing stops there.
- The default NACL allows all traffic in both directions.

Because it is stateless, a typical outbound rule you need is:

```
Allow outbound TCP 1024–65535 to 0.0.0.0/0     ← for replies to inbound connections
```

Most teams leave NACLs at the default and use security groups for control. NACLs are useful as a blunt extra layer — for example, blocking one specific attacking IP range for a whole subnet.

### What is the difference between a Security Group and a NACL?

| | Security Group | Network ACL |
|---|---|---|
| Attached to | A resource (instance, ENI, RDS) | A subnet |
| State | **Stateful** — replies allowed automatically | **Stateless** — you must allow both directions |
| Rules | Allow only | Allow **and** deny |
| Rule order | All rules evaluated together | Numbered, first match wins |
| Can reference another security group | Yes | No, IP ranges only |
| Applies to | Only the resources it is attached to | Everything in the subnet |
| Default | Deny inbound, allow outbound | Allow everything |

**The most important practical point:** the stateless behaviour of NACLs is the source of many confusing problems. A connection works in one direction and fails in the other, or it works from inside the subnet but not from outside. When traffic is blocked and the security group is clearly correct, check the NACL next, in both directions.

Order of evaluation for inbound traffic:

```
Internet  →  Route table  →  NACL (subnet)  →  Security Group (resource)  →  Instance
```

All three must allow it.

### What is VPC Peering?

A direct private connection between two VPCs. Traffic goes over the AWS network, never over the internet.

It works between VPCs in different accounts and different regions.

**Two limits that matter a lot:**

1. **No transitive routing.** If A is peered with B, and B is peered with C, then **A cannot reach C**. Peering is only ever between two VPCs. To connect ten VPCs to each other you would need 45 peering connections, which is unmanageable.

2. **CIDR blocks must not overlap.** If both VPCs use `10.0.0.0/16`, you cannot peer them. There is no NAT option to work around it.

You must also add routes on **both** sides, and update the security groups. Creating the peering connection alone does nothing.

For more than a few VPCs, use a Transit Gateway instead.

### What is a Transit Gateway?

A central hub that connects many VPCs, VPN connections, and Direct Connect links together.

It solves the peering problem. Instead of connecting every VPC to every other VPC, everything connects once to the Transit Gateway.

```
Peering (mesh):              Transit Gateway (hub):

VPC A ─── VPC B                VPC A ─┐
  │  ╲   ╱  │                  VPC B ─┤
  │   ╳     │                  VPC C ─┼── TGW ── VPN
VPC C ─── VPC D                VPC D ─┘
(6 connections for 4)          (4 connections for 4)
```

What it adds over peering:

- **Transitive routing works.** A can reach C through the hub.
- Connects VPN and Direct Connect too, not just VPCs
- **Route tables** on the Transit Gateway let you control which VPCs can reach which. So you can isolate environments, or force all traffic through a shared inspection VPC.
- Scales to thousands of connections

The cost: an hourly charge per attachment, plus a charge per gigabyte. Peering has no hourly charge, so peering is cheaper for a small number of VPCs.

### What is an Elastic IP?

A static public IPv4 address that you own and can move between resources.

A normal public IP on an EC2 instance changes when you stop and start the instance. An Elastic IP does not — it stays the same until you release it.

Used for:

- NAT Gateways (which require one)
- Anything that needs a fixed IP for someone else's firewall rules
- Failover — you can move an Elastic IP to a standby instance

Cost note: AWS now charges for **all** public IPv4 addresses, including Elastic IPs that are in use. There is also an extra charge for an Elastic IP that is allocated but **not** attached to anything. So unattached Elastic IPs are a common source of small wasted spend.

Better alternatives where possible: use a load balancer DNS name instead of a fixed IP, or use an NLB, which gives you a static IP per availability zone.

### What is AWS PrivateLink?

A way to reach a service privately, without going over the internet, without peering, and without exposing whole networks to each other.

How it works: the service provider creates an endpoint service. You create an **interface endpoint** in your VPC. That endpoint appears as a normal network interface with a private IP inside your subnet. You connect to that private IP, and traffic reaches the provider's service.

Why it is useful:

- Traffic never touches the internet
- **CIDR overlap does not matter**, unlike peering
- It is **one-way and one-service**. You expose exactly one service, not your whole VPC. The provider cannot reach into your network.
- It works across accounts, which makes it good for SaaS vendors and for shared internal services

PrivateLink is also the technology behind interface VPC endpoints for AWS services.

### What is a VPC Endpoint?

A way to reach an AWS service privately from inside your VPC, without going through an Internet Gateway or NAT Gateway.

Two reasons to use them:

1. **Security.** Traffic to S3 or ECR stays inside the AWS network.
2. **Cost.** Traffic does not pass through the NAT Gateway, so you do not pay the NAT data processing charge. For high-volume S3 or ECR traffic this saves a lot of money.

There are two types, and the difference matters.

### What is the difference between a Gateway Endpoint and an Interface Endpoint?

| | Gateway Endpoint | Interface Endpoint |
|---|---|---|
| Services supported | **Only S3 and DynamoDB** | Most AWS services |
| How it works | A route in your route table | An ENI with a private IP in your subnet |
| Cost | **Free** | Hourly charge per endpoint per AZ, plus data charge |
| DNS | Uses the normal public DNS name | Gets a private DNS name (private DNS can be enabled) |
| Reachable from on-premises via VPN | **No** | Yes |
| Uses an IP in your subnet | No | Yes, one per AZ |

**A Gateway Endpoint** adds a route. Traffic destined for S3 goes to the endpoint instead of the internet. Because it is free, you should almost always create one for S3 in every VPC.

**An Interface Endpoint** puts a real network interface in your subnet, built on PrivateLink. You use it for services like ECR, Secrets Manager, SSM, CloudWatch Logs, KMS, and many others.

**One very important practical detail for containers:**

To pull an image from ECR privately, you need **three** endpoints, not two:

```
ecr.api        (interface endpoint)   — for authentication and API calls
ecr.dkr        (interface endpoint)   — for the Docker registry protocol
S3             (GATEWAY endpoint)     — because the image LAYERS are stored in S3
```

People often create the two ECR endpoints, then find that image pulls still fail. The reason is the missing S3 gateway endpoint. The ECR API works but the actual layer download fails. Since the S3 gateway endpoint is free, always add it.

### How does an ALB differ from an NLB?

| | ALB (Application Load Balancer) | NLB (Network Load Balancer) |
|---|---|---|
| Layer | 7 (HTTP) | 4 (TCP / UDP / TLS) |
| Protocols | HTTP, HTTPS, gRPC, WebSocket | TCP, UDP, TLS |
| Routing by path or host | **Yes** | No |
| TLS termination | Yes | Yes, or pass through |
| Static IP address | No, DNS name only | **Yes**, one per AZ |
| Preserves the client source IP | No (adds `X-Forwarded-For`) | **Yes** |
| Performance | Very good | Extremely high, millions of requests per second |
| Latency | Slightly higher | Very low |
| WAF support | Yes | No |
| Fixed idle timeout | 60 seconds by default, configurable | 350 seconds, **not** configurable |
| Health checks | HTTP or HTTPS | TCP, HTTP, or HTTPS |

**Choose an ALB when** you have normal web traffic and want path-based routing, host-based routing, redirects, authentication, or a WAF. This covers most web applications.

**Choose an NLB when** you need any of these:

- A protocol that is not HTTP (a database, a message broker, a game server)
- A **static IP address**, because someone else's firewall needs to allow it
- The real client IP without relying on headers
- Extreme performance or very low latency
- End-to-end encryption with no decryption at the load balancer

Two details worth remembering:

- The NLB's 350-second idle timeout **cannot be changed**. If your application's TCP keepalive is longer than that, connections are silently dropped and you get mysterious hangs. Set keepalive below 350 seconds.
- There is also a **Gateway Load Balancer**, used for sending traffic through third-party security appliances, and a **Classic Load Balancer**, which is the old one and should not be used for new work.

---

## Troubleshooting Scenarios

### A website is not accessible. How do you troubleshoot?

Test each layer in order. Each test tells you whether to keep going or stop.

```bash
# 1. Does the name resolve?
dig example.com +short

# 2. Can you reach the port?
nc -zv example.com 443

# 3. Does TLS work?
openssl s_client -connect example.com:443 -servername example.com </dev/null

# 4. Does HTTP work?
curl -v https://example.com

# 5. Where does the time go?
curl -w "dns:%{time_namelookup} conn:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n" -o /dev/null -s https://example.com
```

Then read the result:

| Where it fails | What it means |
|---|---|
| DNS returns nothing | DNS problem — record missing, or resolver broken |
| Port **refused** | You reached the server. Nothing is listening on that port. |
| Port **times out** | Firewall, security group, NACL, or a missing route |
| TLS fails | Certificate problem — expired, wrong name, or incomplete chain |
| HTTP returns 5xx | The server is reachable. The application is failing. |
| HTTP returns 502 / 504 | The load balancer is up, but the backend is down or too slow |
| Works from one place but not another | DNS caching, geographic routing, or a regional problem |

Also check whether it is broken for everyone or just for you. Test from a different network or use an external checking service. If it works from outside your network, the problem is on your side.

### DNS resolution is failing. What do you check?

```bash
cat /etc/resolv.conf              # 1. which DNS server am I using?
dig example.com                   # 2. what does it say? check the status line
dig @8.8.8.8 example.com          # 3. compare with a public resolver
dig NS example.com +short         # 4. who is authoritative?
dig @ns1.provider.com example.com # 5. ask the authoritative server directly
dig +trace example.com            # 6. see the full path
nc -zvu 8.8.8.8 53                # 7. can I even reach port 53?
```

Reading the `dig` status:

| Status | Meaning | What to do |
|---|---|---|
| `NXDOMAIN` | The name does not exist | Check for a typo, or a missing record |
| `SERVFAIL` | The DNS server failed | Often DNSSEC, or a broken upstream |
| `REFUSED` | The server will not answer you | You are not allowed to query it |
| `NOERROR`, no answer section | The name exists, but not with that record type | Ask for the right type (A, AAAA, CNAME) |
| No response at all | Port 53 is blocked | Check the firewall |

**The most useful single step** is comparing your resolver with `8.8.8.8`. If the public one is correct and yours is wrong, the problem is your resolver or its cache. If both are wrong, the problem is the authoritative record.

If the authoritative server has the right answer but resolvers give the old one, it is caching. You have to wait for the TTL.

### ping works but HTTP doesn't. Why?

This tells you a lot. `ping` uses ICMP at layer 3. HTTP uses TCP at layer 4 and above. So the network path is fine, and the problem is higher up.

The possible causes:

| Cause | How to confirm |
|---|---|
| Nothing is listening on the port | `nc -zv host 80` gives **refused** |
| A firewall blocks that port | `nc -zv host 80` **times out** |
| The application crashed | Check the process on the server: `ss -tulnp` |
| The app is bound to `127.0.0.1` only | `ss -tulnp` shows `127.0.0.1:80` instead of `0.0.0.0:80` |
| Wrong port | You are testing 80 but the app listens on 8080 |
| The app is up but very slow | The connection succeeds, then hangs |
| TLS problem (for HTTPS) | TCP works, `openssl s_client` fails |
| The web server is running but misconfigured | You get a 502 or 503 rather than nothing |

The two commands that separate these:

```bash
nc -zv host 80         # is the port reachable?
ss -tulnp | grep :80   # on the server: is anything listening, and on which address?
```

The bind address is a very common cause. `127.0.0.1:80` works when you test on the machine and fails from anywhere else.

### The application is timing out. What are the possible causes?

A timeout means something waited and got no answer. Work through the layers.

**Network level:**
- A firewall silently dropping packets (a drop causes a timeout; a reject causes "refused")
- A missing route
- **MTU mismatch** — small requests work, large ones hang
- Packet loss causing retransmissions and long delays

**Connection level:**
- The server is not listening
- The listen backlog is full because the app cannot accept connections fast enough
- Ephemeral ports exhausted on the client
- Connection pool exhausted, so requests wait for a free connection

**Application level:**
- A slow database query
- Waiting on another API that is slow
- Thread pool full, so requests queue before processing starts
- A long garbage collection pause
- Lock contention
- CPU saturated, or CPU throttling in a container

**Configuration level:**
- The timeout is simply too short for that operation
- Timeouts do not fit together — the outer timeout is shorter than the inner one, so the outer gives up while work continues
- A load balancer idle timeout shorter than the app's keepalive

**How to narrow it down quickly:**

```bash
curl -w "dns:%{time_namelookup} conn:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer}\n" -o /dev/null -s https://example.com
```

If `conn` never completes, it is network or firewall. If `conn` is fast but `ttfb` is slow, the network is fine and the application is slow. That single output splits the problem in half.

### One server can reach the database, another cannot. How do you investigate?

One works and one does not, so compare them. The difference is the answer.

```bash
# Run on BOTH servers and compare the output

nslookup db.example.com          # does the name resolve to the same address?
nc -zv db.example.com 5432       # can it reach the port?
ip route get <db-ip>             # which route and interface is used?
ip addr                          # what is my own IP and subnet?
env | grep -i db                 # is the configuration the same?
```

Things that commonly differ:

| Difference | How to check |
|---|---|
| Different subnet, different route table | Which subnet is each server in? |
| Different security group | Does the database allow **both** source security groups? |
| Different NACL, because different subnet | Check both directions on the working and broken subnet |
| Stale DNS cache on one server | Compare `nslookup` output on both |
| Different credentials or config | Compare environment variables and config files |
| Clock skew on one server | `timedatectl` — breaks TLS and tokens |
| One is at its connection limit | Check `pg_stat_activity` for connections from each host |

**The most likely cause on AWS:** the two servers are in different subnets or have different security groups, and the database only allows one of them. Check the database's inbound rules and confirm both source security groups are listed.

### A Pod cannot connect to an external API. What do you check?

Test from inside the Pod, step by step:

```bash
kubectl exec -it <pod> -- sh

# inside the pod:
nslookup api.example.com          # 1. DNS
nc -zv api.example.com 443        # 2. can it reach the port?
curl -v https://api.example.com   # 3. does the full request work?
env | grep -i proxy               # 4. is a proxy configured?
```

Then check outward:

**1. DNS.** If `nslookup` fails, it is a DNS problem, not a connectivity problem. Check CoreDNS, and check whether a NetworkPolicy is blocking port 53.

**2. NetworkPolicy egress.** A deny-all egress policy blocks outbound traffic. This is a very common cause.

```bash
kubectl get networkpolicy -n <namespace>
```

**3. Is there a route to the internet?** For a Pod in a private subnet, all of this must be true:

- A NAT Gateway exists in a public subnet
- The private subnet's route table sends `0.0.0.0/0` to the NAT Gateway
- The NAT Gateway is healthy and in a working AZ

**4. Security group.** Does the node's or Pod's security group allow outbound on 443? Outbound is usually allowed by default, but it may have been restricted.

**5. NACL.** Stateless, so check that replies are allowed back in on the ephemeral port range.

**6. Is the external API blocking you?** Check whether they allow-list IP addresses. Your NAT Gateway IP may not be on their list. Also check for rate limiting — you would get 429 rather than a timeout.

**7. A proxy.** If your cluster requires an HTTP proxy for egress, the Pod needs `HTTP_PROXY` and `HTTPS_PROXY` set, plus `NO_PROXY` for internal addresses.

### A Kubernetes Service is unreachable. How do you troubleshoot?

Check in this order. The first two find most problems.

```bash
# 1. Does the Service have endpoints?  ← check this first
kubectl get endpointslices -l kubernetes.io/service-name=<svc>

# 2. Do the labels match?
kubectl get svc <svc> -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels

# 3. Are the Pods Ready?
kubectl get pods -l app=<label>

# 4. Are the ports right?
kubectl get svc <svc> -o yaml | grep -A3 ports
kubectl get pod <pod> -o yaml | grep -A3 containerPort

# 5. Test from inside the cluster
kubectl run test --rm -it --image=nicolaka/netshoot -- sh
#   curl <service>.<namespace>:<port>
#   curl <pod-ip>:<port>
```

Read the result:

| Finding | Cause |
|---|---|
| No endpoints | Selector does not match, or no Pod is Ready |
| Endpoints exist, Pod IP works, Service name fails | DNS problem |
| Endpoints exist, Pod IP also fails | The application is not working, or wrong port |
| Service works inside the cluster, not outside | Ingress, load balancer, or NodePort problem |
| Some requests work, some fail | One unhealthy Pod is still in the endpoint list |

`targetPort` is worth checking carefully. If the Service says `targetPort: 8080` and the container listens on 3000, the Service has endpoints and still nothing works.

### An ALB returns 502 Bad Gateway. What could be wrong?

502 means the ALB reached your target but did not get a valid HTTP response.

Causes:

| Cause | How to check |
|---|---|
| The target closed the connection early | Check the app logs for crashes |
| The app returned something that is not valid HTTP | Test the target directly with `curl -v` |
| **The app's keepalive timeout is shorter than the ALB's idle timeout** | Very common. Set the app's keepalive **higher** than the ALB's 60 seconds. |
| The app crashed while handling the request | Check for out-of-memory kills, and stack traces |
| Wrong protocol — HTTP sent to an HTTPS port, or the reverse | Check the target group protocol setting |
| The response header was too large | Check for very large cookies or headers |
| TLS mismatch when the target group uses HTTPS | Certificate on the target is invalid |

**The keepalive one deserves attention** because it is common and confusing. If the app closes an idle connection at 5 seconds and the ALB keeps it for 60, the ALB will eventually reuse a connection the app has already closed. The result is intermittent 502 errors with no pattern and nothing in the app logs.

The rule: **the target's keepalive timeout must be longer than the ALB idle timeout.**

Where to look:

```
ALB access logs  →  elb_status_code = 502, target_status_code = "-"
```

A `target_status_code` of `-` means the target never sent a valid response at all. That confirms it is a connection-level problem rather than an application error.

### An ALB returns 504 Gateway Timeout. What could be wrong?

504 means the target did not answer within the ALB's idle timeout (60 seconds by default).

Causes:

| Cause | How to check |
|---|---|
| The app is genuinely slow for this request | Check latency per endpoint, at p95 and p99 |
| A slow database query | Check the slow query log |
| A slow call to another service | Check downstream latency |
| Thread pool or connection pool full | Requests queue before processing starts |
| The idle timeout is too short for this operation | Compare the operation's normal time with the timeout |
| The app is overloaded | Check CPU, memory, and CPU throttling |
| A deadlock or a long lock wait | Check the app's thread state |

A useful clue: **is it all requests, or only some?**

- **Only one endpoint** — that operation is slow. Either make it faster, make it asynchronous, or raise the timeout for that path.
- **All endpoints** — the whole application is overloaded or a shared dependency is slow.
- **Random requests** — some Pods or instances are unhealthy, or there is a resource limit being hit intermittently.

To confirm the ALB is not the problem, time the target directly:

```bash
curl -w "%{time_total}\n" -o /dev/null -s http://<target-ip>:<port>/path
```

If the target is also slow, the ALB is reporting the truth.

### Users report intermittent network failures. How do you investigate?

"Intermittent" means it is correlated with something you have not identified yet. Find the pattern before forming a theory.

**Check each dimension:**

| Dimension | Question |
|---|---|
| Which server | Is it always the same instance or Pod? One bad backend out of five gives exactly 20% failures. |
| Which AZ | Is it one availability zone? |
| Which client | One region, one ISP, one office? |
| Which time | Is it correlated with deploys, cron jobs, scaling events, or backups? |
| Which request | One endpoint, large payloads, or long-running requests? |
| Which load level | Does it happen only at peak? Then it is saturation. |

**Common causes of intermittent specifically:**

- **One unhealthy backend still receiving traffic.** The failure rate matches `1 ÷ number of backends`.
- **DNS timeouts.** In Kubernetes, the conntrack race gives roughly 5-second delays.
- **Connection pool exhaustion** at peak load.
- **Idle timeout mismatch** between the load balancer and the application (see the 502 answer).
- **Packet loss** — a small percentage causes big latency jumps because TCP retransmits.
- **Ephemeral port or conntrack table exhaustion** on a NAT gateway or node.
- **Requests hitting a Pod that is shutting down**, during a rollout.
- **Rate limiting** kicking in above a threshold.

**What actually finds it:**

```bash
mtr --report --report-cycles 300 <host>    # sustained loss per hop
ss -ti                                     # retransmissions per connection
netstat -s | grep -i -E 'retrans|drop'     # system totals
```

The important measurement point: **look at per-instance and per-endpoint metrics, not averages.** An average dashboard hides a 2% failure rate completely. You need to break the metrics down by Pod, by node, and by AZ, or you will not see it.

### High network latency is observed. What metrics and tools would you use?

**First, split the time up.** This is the highest-value step, because it tells you which stage is slow.

```bash
curl -w "dns:%{time_namelookup} conn:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n" \
     -o /dev/null -s https://example.com
```

| Which number is high | Where the problem is |
|---|---|
| `dns` | DNS resolution |
| `conn` | TCP connection setup — network distance or packet loss |
| `tls` | TLS handshake — certificate chain, or an extra round trip |
| `ttfb` | The **server** is slow. Not the network. |
| `total` much higher than `ttfb` | Transferring the body is slow — bandwidth or a large response |

**Then, tools by layer:**

| Layer | Tools |
|---|---|
| Path and loss | `mtr`, `traceroute`, `ping` |
| TCP details | `ss -ti` (round trip time, retransmits), `tcpdump` |
| Interface | `ip -s link` (errors, drops), `ethtool -S eth0` |
| System | `netstat -s`, `sar -n DEV 1` |
| Application | Distributed tracing, application metrics |
| Cloud | CloudWatch metrics for the ALB, NLB, and NAT Gateway |

**Metrics to look at:**

- Latency at **p50, p95 and p99**, never the average
- TCP retransmission rate
- Packet loss per hop
- Connection count and connection setup rate
- Bandwidth against the instance's limit
- CPU **throttling** on containers, which looks exactly like network latency
- Cross-AZ traffic volume, since each hop adds a small amount of latency

**Causes people miss:**

1. **It is not the network.** Most of the time high "network latency" is a slow database, a garbage collection pause, or CPU throttling. Confirm with the `ttfb` number first.
2. **No keepalive**, so every request pays for a TCP and TLS handshake.
3. **Cross-AZ or cross-region traffic** adding a few milliseconds each way, many times per request.
4. **conntrack table full**, causing dropped packets and retries.

### Packets are being dropped. How do you debug it?

Find **where** they are dropped. Check from the closest point outward.

**1. On the local machine:**

```bash
ip -s link                          # interface errors and drops
ethtool -S eth0 | grep -i -E 'drop|err|discard'
netstat -s | grep -i -E 'drop|prune|overflow|listen'
```

`netstat -s` is worth reading carefully. Lines like `listen queue overflow` or `packets pruned from receive queue` tell you the machine is dropping packets because it cannot keep up — a kernel buffer or backlog problem, not a network problem.

**2. Connection tracking table full.** A common and easily missed cause:

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
dmesg | grep -i conntrack        # look for "table full, dropping packet"
```

When this table is full, new connections are dropped silently. It looks random and has no obvious cause.

**3. On the network path:**

```bash
mtr --report --report-cycles 300 <host>
```

Read the loss column carefully. **Loss on a middle hop only, with no loss at the end, is normal** — that router is just not replying to ICMP. Only loss that continues to the **final hop** is real.

**4. Firewall drops.** Add counters or logging to see if rules are dropping traffic:

```bash
iptables -L -n -v        # look at the packet counters per rule
```

**5. On AWS:**

- **VPC Flow Logs** — look for `REJECT` entries, which show security group or NACL blocks
- The `PacketDropCount` metric on the instance
- Instance-level allowance metrics: `bw_in_allowance_exceeded`, `conntrack_allowance_exceeded`, `pps_allowance_exceeded`. Smaller instance types have low limits and drop packets when exceeded. This is often the answer when nothing else explains it.

**6. MTU problems.** If large packets are dropped and small ones are fine:

```bash
ping -M do -s 1472 <host>      # test full 1500 MTU
ping -M do -s 1400 <host>      # test smaller
```

### A Security Group is correct, but traffic is still blocked. What else would you check?

Go through every other layer that can block traffic.

**1. NACL.** The most likely answer. It is **stateless**, so the reply direction must be allowed separately. Check both inbound and outbound on both subnets involved.

**2. Route table.** A missing route looks exactly like a firewall block — the connection times out. Check that both subnets have a route to each other, and to the internet if needed.

```bash
ip route get <destination-ip>
```

**3. The other side's security group.** You checked yours. Did you check the destination's inbound rules?

**4. The operating system firewall on the instance:**

```bash
iptables -L -n -v
nft list ruleset
ufw status
firewall-cmd --list-all
```

**5. The application's bind address.** If it listens on `127.0.0.1` only, no firewall change will ever help.

```bash
ss -tulnp | grep <port>
```

**6. SELinux or AppArmor.** These can block network access even when the firewall allows it.

```bash
getenforce
ausearch -m avc -ts recent
```

**7. Is the application actually running?** A connection refused means the port is reachable and nothing is listening. That is not a firewall issue at all.

**8. In Kubernetes: NetworkPolicy.** A separate layer from AWS security groups.

```bash
kubectl get networkpolicy -A
```

**9. Wrong protocol or port.** Check that the rule allows TCP where TCP is needed, and covers the right port range. UDP rules do not help TCP traffic.

**The fastest way to see what is actually blocking:** VPC Flow Logs. They show `ACCEPT` and `REJECT` per flow, so you can confirm whether the packet even arrived and which layer rejected it.

### EC2 instances in a private subnet cannot access the internet. How do you troubleshoot?

Check all the required pieces. Every one must be correct.

**1. Does a NAT Gateway exist, and is it available?** Check its state in the console. Confirm which AZ it is in.

**2. Is the NAT Gateway in a PUBLIC subnet?** This is a common mistake. A NAT Gateway placed in a private subnet cannot reach the internet itself, so nothing works.

**3. Does the PRIVATE subnet's route table point to the NAT Gateway?**

```
0.0.0.0/0  →  nat-xxxxxxxx
```

Check you are looking at the route table for the correct subnet. A subnet may be using the main route table instead of the one you edited.

**4. Does the PUBLIC subnet's route table point to the Internet Gateway?**

```
0.0.0.0/0  →  igw-xxxxxxxx
```

**5. Is there an Internet Gateway attached to the VPC?**

**6. Security group outbound rules.** Does the instance allow outbound on 443 and 80? Outbound is allowed by default but may have been restricted.

**7. NACL on both subnets.** Stateless, so:
- Private subnet: allow outbound to the internet, and allow inbound on the ephemeral range for replies
- Public subnet: the same

**8. Is the NAT Gateway in the same AZ?** If the NAT Gateway is in AZ-a and the instance is in AZ-b, traffic still works but crosses AZs, which costs money. Worse, if the NAT Gateway's AZ fails, the instance loses internet access. Best practice is one NAT Gateway per AZ.

**9. Check the NAT Gateway metrics.** `ErrorPortAllocation` means it has run out of ports, which happens with very many connections to the same destination.

**Test from the instance:**

```bash
curl -v --max-time 5 https://checkip.amazonaws.com     # should return the NAT Gateway's IP
dig +short google.com                                  # is DNS working?
ip route get 8.8.8.8                                   # which route is used?
```

If the returned IP is the NAT Gateway's Elastic IP, the path is working. If DNS fails but routing looks right, the problem is DNS, not internet access. Check `enableDnsSupport` on the VPC.

### An EC2 instance cannot be reached via SSH. What are your troubleshooting steps?

**1. Is the instance running and healthy?** Check the console. You want `2/2 checks passed`.

- **System status check failing** — an AWS hardware problem. Stop and start the instance to move it to new hardware.
- **Instance status check failing** — a problem inside the operating system.

**2. Security group.** Is port 22 allowed inbound from your IP address? Note that your home IP may have changed since the rule was written.

**3. NACL.** Stateless. Allow inbound 22, and allow outbound on the ephemeral range for replies.

**4. Route table and public IP.** For a public instance: is there a route to the Internet Gateway, and does the instance actually have a public IP? A stopped and started instance loses a non-elastic public IP.

**5. Is it in a private subnet?** Then you must go through a bastion host or VPN. Check the bastion first.

**6. Read the console output.** This works without any network access and often shows the answer directly:

```bash
aws ec2 get-console-output --instance-id i-xxxxx --output text
```

Look for a full disk, a failed boot, or filesystem errors.

**7. Common causes on the instance itself:**

| Cause | Symptom |
|---|---|
| Disk full | SSH hangs or refuses, because it cannot write session files |
| sshd not running | Connection refused |
| Wrong permissions on `~/.ssh` or `authorized_keys` | Permission denied |
| A bad `sshd_config` change | sshd failed to restart |
| Wrong key or wrong username | Permission denied (publickey) |
| CPU at 100% | Very slow or no response |

**8. Use a route that does not need SSH:**

- **Session Manager (SSM)** — works without port 22, without a public IP, and without a security group rule. It uses an agent on the instance.
- **EC2 Serial Console** — direct console access, useful for boot problems.

**9. Last resort recovery:** stop the instance, detach the root volume, attach it to another working instance, fix the problem (for example, clear the full disk or repair `sshd_config`), then reattach it.

**Get more detail from the client side:**

```bash
ssh -vvv user@host        # verbose output shows exactly which stage fails
```

### A Kubernetes Pod cannot resolve DNS names. How would you debug it?

```bash
# 1. Start a debug pod
kubectl run dnstest --rm -it --image=nicolaka/netshoot -- bash

# inside:
cat /etc/resolv.conf                # is the nameserver correct?
nslookup kubernetes.default         # internal name
nslookup google.com                 # external name
dig @10.96.0.10 kubernetes.default  # ask CoreDNS directly
```

**2. Check CoreDNS:**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100
kubectl get svc -n kube-system kube-dns
kubectl get endpointslices -n kube-system -l kubernetes.io/service-name=kube-dns
```

**3. Read the pattern:**

| Result | Cause |
|---|---|
| Both internal and external fail | CoreDNS is down, or the Pod cannot reach it |
| Internal works, external fails | Upstream forwarding broken, or egress blocked |
| Internal fails, external works | The record does not exist, or wrong search domains |
| Works but takes about 5 seconds | The UDP conntrack race condition |
| Works from one node only | The CNI or kube-proxy is broken on the other node |

**4. Check the most common specific causes:**

**NetworkPolicy blocking port 53.** Very common after adding a deny-all egress policy:

```bash
kubectl get networkpolicy -A
```

You need to allow UDP 53 (and ideally TCP 53) to the `kube-system` namespace.

**Wrong `dnsPolicy`.** If the Pod spec has `dnsPolicy: Default`, it uses the node's DNS and cannot resolve cluster names at all. It should normally be `ClusterFirst`.

**CoreDNS overloaded.** Check its CPU usage and whether it is being throttled. Under load it gets slow and you see intermittent timeouts rather than clean failures.

**conntrack table full on the node**, dropping DNS packets:

```bash
kubectl debug node/<node> -it --image=busybox
# cat /proc/sys/net/netfilter/nf_conntrack_count
```

**5. Fixes for the 5-second delay problem:** install NodeLocal DNSCache, set `ndots: 2` in the Pod's DNS config, or use TCP for DNS.

### Your application works locally but fails after deployment. How do you approach the network troubleshooting?

Locally, everything is on one machine. After deployment, everything is separated by a network. So look for every assumption that only holds on one machine.

**Check these, in order of how often they are the cause:**

**1. Bind address.** The most common cause.

Locally the app binds to `127.0.0.1` and it works, because the client is also local. In a container or on a server, `127.0.0.1` means "only this container", so nothing outside can reach it.

```bash
ss -tulnp | grep <port>
# 127.0.0.1:8080   ← wrong, will not work
# 0.0.0.0:8080     ← correct
```

**2. Hostnames.** Locally you use `localhost:5432` for the database. After deployment the database is a different machine. Every `localhost` in your configuration must become a real hostname or Service name.

**3. Environment variables and secrets.** Are they all actually set in the deployed environment? A missing variable often makes the app fall back to a local default.

```bash
kubectl exec <pod> -- env | sort
```

**4. DNS.** Can the deployed app resolve the names it needs? A container has different DNS settings than your laptop.

**5. Firewall and security groups.** Locally there is no firewall between components. In production there are several layers. Check security groups, NACLs, and NetworkPolicies.

**6. Ports and port mapping.** Is the container port exposed? Does the Service `targetPort` match the port the app really listens on?

**7. TLS and certificates.** Locally you may use plain HTTP or skip verification. In production TLS is enforced. Check that the container has CA certificates installed — a minimal base image may not have them, so every HTTPS call fails with a certificate error.

**8. Egress rules.** Can the app reach the internet at all? A Pod in a private subnet needs a NAT Gateway. A deny-all egress policy blocks everything.

**Test from inside the deployed environment**, not from your laptop:

```bash
kubectl exec -it <pod> -- sh
# then test DNS, ports and the actual calls from there
```

That last point matters. Testing from your laptop tells you about your laptop's network. The Pod's network is different.

### How would you troubleshoot SSL/TLS certificate errors?

**First, read the error.** Each error points at a different check, so the message narrows it immediately.

| Error | Cause |
|---|---|
| `ERR_CERT_DATE_INVALID` / `certificate has expired` | Expired, or the client clock is wrong |
| `ERR_CERT_COMMON_NAME_INVALID` / `hostname mismatch` | The certificate is not valid for this name |
| `ERR_CERT_AUTHORITY_INVALID` / `unable to get local issuer certificate` | The chain is incomplete, usually a missing intermediate |
| `SSL_ERROR_NO_CYPHER_OVERLAP` / `handshake failure` | No shared TLS version or cipher |
| `ERR_CERT_REVOKED` | The certificate was revoked |

**Then inspect the certificate:**

```bash
# Full handshake details, including the chain
openssl s_client -connect example.com:443 -servername example.com

# Just the important fields
openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -dates -subject -issuer

# All the names the certificate covers
openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -text | grep -A1 "Subject Alternative Name"
```

**The four checks and how to fix each:**

**1. Dates.** Compare `notAfter` with today. If expired, renew it. Then find out why automatic renewal did not work — that is the real problem. For AWS ACM, check that the DNS validation record still exists; if someone deleted it, renewal fails silently.

**2. Name.** Does the hostname appear in the Subject Alternative Name list? A wildcard `*.example.com` covers `api.example.com` but **not** `example.com` itself, and not `a.b.example.com`. That surprises people.

**3. Chain.** Look at the `Certificate chain` section of the `s_client` output. If it shows only your certificate and no intermediate, the chain is incomplete.

This is the one that causes confusing bugs: browsers can sometimes fetch the missing intermediate themselves, so the site looks fine in Chrome but fails in `curl`, in Java applications, and on mobile. Fix it by configuring the server to send the full chain (the "fullchain" file, not just the certificate).

```bash
openssl s_client -connect example.com:443 -showcerts    # see every certificate sent
```

**4. Client trust store.** If the certificate is fine but one client rejects it, that client may lack the CA certificates. Minimal container images often have no `ca-certificates` package at all. Install it.

**Two other things to check:**

- **Which layer is serving the certificate?** With CloudFront in front of an ALB, each can have its own certificate. You may fix the ALB while the expired one is on CloudFront. Check which layer you actually reached.
- **SNI.** If you test without SNI, you get the server's default certificate, which is usually the wrong one. Always use `-servername`.

Never leave `curl -k` or "skip verification" in production. It is fine for a quick test and it removes the protection entirely.

### Users complain that only some requests fail while others succeed. How do you investigate?

Partial failure means the requests take different paths, or hit different backends. Find what is different.

**1. Calculate the failure rate.** This alone often identifies the cause.

If exactly 25% of requests fail and you have 4 backends, **one backend is broken**. If it is 33% with 3 backends, the same. That maths points straight at the answer.

**2. Break the metrics down.** Averages hide this problem completely. Group your error rate by:

- Backend instance or Pod
- Node
- Availability zone
- Endpoint
- Client region
- Request size

Whichever grouping shows the errors concentrated is your answer.

**3. Check for one unhealthy backend still receiving traffic:**

```bash
# Kubernetes
kubectl get pods -l app=<label>                  # is any Pod not Ready but still listed?
kubectl get endpointslices -l kubernetes.io/service-name=<svc>

# AWS
# Check target group health in the console — look for a target that flaps
```

A Pod that flaps between Ready and not Ready is the classic cause. It stays in the endpoint list part of the time.

**4. Check for size or content differences.** If large requests fail and small ones succeed:

- MTU mismatch
- A body size limit on the proxy (`client_max_body_size` in NGINX)
- A timeout that only long requests exceed

**5. Check for timing.** Do failures line up with:

- Deploys (requests hitting a Pod that is shutting down)
- Autoscaling events (new instances not warm yet)
- Cron jobs or batch loads
- Peak traffic (which means saturation)

**6. Check the load balancer access logs.** These are the best source, because every request is recorded with its target:

```
# ALB access log fields to look at:
target_status_code       ← what the backend returned
elb_status_code          ← what the user got
target_processing_time   ← how long the backend took
target:port              ← WHICH backend served it
```

Group the failures by the `target:port` field. If they all have the same target, you have found the broken backend in one query.

**7. Other common causes of partial failure:**

- Rate limiting, which returns 429 above a threshold
- Sticky sessions sending some users to a bad server
- A cache with some poisoned entries
- DNS returning several IPs where one is dead
- Connection pool exhaustion, which only affects requests during peak

### Which networking tools would you use in different scenarios?

| Tool | Use it for |
|---|---|
| `ping` | Basic reachability and round trip time. Remember ICMP is often blocked. |
| `traceroute` | Find the path and which hop is slow. Use `-T -p 443` when ICMP is blocked. |
| `mtr` | Better than both — continuous, shows loss per hop. Best for intermittent problems. |
| `dig` | Anything DNS. Comparing resolvers, checking records, `+trace` for the full path. |
| `nslookup` | Simple DNS check. `dig` is better but `nslookup` exists on Windows too. |
| `curl` | HTTP testing, headers, status codes, and the timing breakdown with `-w`. |
| `nc` (netcat) | Test whether a TCP or UDP port is reachable. Distinguishes refused from timeout. |
| `ss` | Sockets, listening ports, connection states, and per-connection stats with `-ti`. |
| `netstat` | Older version of `ss`. Still useful for `netstat -s` protocol counters. |
| `tcpdump` | Capture actual packets when you need proof of what was sent or received. |
| `ip` | Addresses (`ip addr`), routes (`ip route`), ARP (`ip neigh`), interface stats (`ip -s link`). |
| `openssl s_client` | Inspect certificates and debug TLS handshakes. |
| `nmap` | Scan which ports are open on a host. Get permission before using it. |
| `iperf3` | Measure actual bandwidth between two machines. |
| `ethtool` | Interface-level statistics, errors, and link speed. |

**Mapping problems to tools:**

| Problem | Reach for |
|---|---|
| Site not loading | `dig` → `nc` → `curl -v` |
| Slow response | `curl -w` first, then `mtr` and `ss -ti` |
| DNS problem | `dig` against several resolvers, then `dig +trace` |
| Port seems blocked | `nc -zv` (refused vs timeout), then `ss -tulnp` on the server |
| Certificate error | `openssl s_client -servername` |
| Packet loss | `mtr --report`, then `ip -s link` and `netstat -s` |
| Intermittent failures | Load balancer access logs, `mtr` over a long period, per-instance metrics |
| Need proof of what happened | `tcpdump -w` and then Wireshark |
| Routing question | `ip route get <destination>` |
| Bandwidth question | `iperf3` |

**Three habits that save the most time:**

1. **`curl -w` before anything else** for a slow request. It tells you immediately whether the problem is DNS, connection, TLS, or the server.
2. **`nc -zv` to tell refused from timeout.** Refused means you reached the machine and nothing is listening. Timeout means a firewall dropped the packet. Two completely different investigations.
3. **`mtr` instead of `ping` or `traceroute`** for anything intermittent. A single `ping` run can easily miss 1% packet loss.
