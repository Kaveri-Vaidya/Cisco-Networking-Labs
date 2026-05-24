# Life of a Packet

## What this lab is about
This lab is all about taking the theoritical knowledge of how DNS works and what ARP does to a practical level! It's about configuring routers to resolve 
hostnames and then watching which MAC addresses appear in 
which ARP caches. This made both concepts significantly more 
concrete. Theory said ARP is link-local. Seeing R3 absent 
from R1's ARP cache after a successful ping to it was the 
moment that actually landed.

---

## Topology
```
[DNS-Server]
     10.10.10.10
          |
         F0/3
          |
F0/1    SW1    F0/2
 |               |
F0/0            F0/0
 |               |
R1           10.10.10.2
10.10.10.1        R2
|
F1/0 ── 10.10.20.2
|
F0/2  SW2  F0/1
|
F0/0
|
10.10.20.1
R3
```
Static routes between R1 and R3 was configured using staic routes [demonstrated in lab 2](../2.%20cisco-device-functions). The focus here will be DNS and ARP.

You can refer the .pkt file to understand the DNS configuration.

---

## Field Notes

### Configuring Routers as DNS Clients

Two commands needed on every router — both matter:
R1(config)#ip domain-lookup
R1(config)#ip name-server 10.10.10.10
`ip domain-lookup` — tells the router to attempt DNS 
resolution when it encounters a hostname it doesn't 
recognise. Without this the router doesn't even try.

`ip name-server` — tells the router *where* to send 
DNS queries. Without this the router doesn't know who to ask.

Both commands are needed. One without the other does nothing.

### What Successful DNS Resolution Looks Like

Before DNS was configured, pinging by hostname failed 
immediately. After configuration the output changed:
R1#ping R2
Translating "R2"...domain server (10.10.10.10)
Sending 5, 100-byte ICMP Echos to 10.10.10.2
!!!!!
Success rate is 100 percent (5/5)

That `Translating "R2"` line is the DNS query happening 
in real time — the router contacted the server, received 
10.10.10.2 back, and used it as the ping destination. 
This line also shows which DNS server was used, which is 
useful for catching misconfigurations immediately.

### ARP Cache — The Most Interesting Discovery

After pinging across the topology I checked R1's ARP cache 
expecting to see every device I had communicated with:
```text
R1#show arp
Protocol  Address      Age  Hardware Addr    Type  Interface
Internet  10.10.10.1   -    0090.0CD7.0D01   ARPA  FastEthernet0/0
Internet  10.10.10.2   4    0004.9A96.A9A5   ARPA  FastEthernet0/0
Internet  10.10.10.10  2    0090.21C6.D284   ARPA  FastEthernet0/0
```

R3 wasn't there — even though I had just successfully 
pinged it. This is where the theory became real:

ARP is a broadcast protocol. Broadcasts don't cross 
router boundaries. So R1 never ARPs for R3 directly — 
it ARPs for R2 (the next hop toward R3) and R2 handles 
the rest from there.

R1's ARP cache shows only what's on its directly 
connected network — 10.10.10.0/24. Everything beyond 
that is R2's problem.

R2 sitting at the boundary of both networks showed 
the full picture:
```text
R2#show arp
Protocol  Address      Age  Hardware Addr    Type  Interface
Internet  10.10.10.1   4    0090.0CD7.0D01   ARPA  FastEthernet0/0
Internet  10.10.10.2   -    0004.9A96.A9A5   ARPA  FastEthernet0/0
Internet  10.10.10.10  1    0090.21C6.D284   ARPA  FastEthernet0/0
Internet  10.10.20.1   4    0030.F2BA.30E7   ARPA  FastEthernet1/0
Internet  10.10.20.2   -    0060.2FCA.ACA0   ARPA  FastEthernet1/0
```
R2 sees both networks entirely because it's directly 
connected to both. R1 and R3 each only see their own 
side of the network.

The `-` in the Age column means that's the router's 
own IP address — age doesn't apply to yourself.

### ARP Cache Summary — Who Sees What

| Router | Directly Connected Networks | ARP Entries For |
|--------|---------------------------|-----------------|
| R1 | 10.10.10.0/24 only | R2, DNS server, itself |
| R2 | 10.10.10.0/24 and 10.10.20.0/24 | Everyone |
| R3 | 10.10.20.0/24 only | R2's far interface, itself |

---

## Arsenal
```text
ip domain-lookup              → enable DNS resolution on router
ip name-server [ip]           → point router to a DNS server
show arp                      → view the ARP cache
show ip interface brief       → verify interface status and IPs
ping [hostname]               → test DNS resolution and connectivity
```
---

## Traps I Fell Into

- Made a couple of DNS configuration mistakes during this lab 
  that taught me more than the working configuration did. 
  Documented in detail here →
  [DNS Resolution Failure](../../troubleshooting-logs/dns-resolution-failure.md)

- The `-` in the ARP age column looks like an error at first. 
  It just means that entry is the router's own interface — 
  there's no age because the router doesn't learn its own 
  address from the network.

---

## Onwards To
[Routing Fundamentals](../16-routing-fundamentals/README.md)
