# DNS Resolution Failure — Troubleshooting Log

## The Situation
While configuring routers as DNS clients in my [Life of a Packet](../labs/3.%20life-of-a-packet) topology 
(R1, R2, R3 with a DNS server at 10.10.10.10), I ran into a 
situation where DNS simply wasn't resolving. I knew the theory 
— routers need a name-server configured and domain-lookup 
enabled — but getting it working cleanly in practice was a 
different story.

---

## Topology at the Time
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
---

## What I Was Trying to Do
Configure R1, R2 and R3 to resolve hostnames using the DNS 
server at 10.10.10.10 — so that instead of pinging by IP 
address I could just ping by hostname like `ping R2`.

The configuration I needed on each router was:
ip domain-lookup
ip name-server 10.10.10.10

Sounds straightforward. It wasn't.

---

## Mistake 1 — Skipped `ip domain-lookup`

My first attempt on R3:
R3(config)#ip name-server 10.10.10.10

Tested it:
R3#ping R1
% Unrecognized host or address, or protocol not running.

Sat there confused for a bit. The name-server was configured, 
connectivity to the DNS server was fine — so why wasn't it 
resolving?

Turns out `ip domain-lookup` is what actually tells the router 
to attempt DNS resolution in the first place. Without it the 
router doesn't even try — it just immediately fails. The 
name-server entry alone does nothing if lookup is disabled.

Fix:
R3(config)#ip domain-lookup
R3(config)#ip name-server 10.10.10.10

Lesson learned — two commands needed, not one. The name-server 
tells the router *where* to ask. The domain-lookup tells the 
router *to ask at all*.

---

## Mistake 2 — Typed the Wrong DNS Server IP on R1

Configured R1 in a hurry and typed:
R1(config)#ip domain-lookup
R1(config)#ip name-server 10.10.10.1

Note the last octet — typed `.1` instead of `.10`. That's R1's 
own IP address, not the DNS server.

Tested:
R1#ping R2
Translating "R2"...domain server (10.10.10.1)
% Unrecognized host or address, or protocol not running.

The error output actually told me exactly what was wrong — 
`domain server (10.10.10.1)` — the wrong IP was sitting right 
there in the output. I nearly read past it.

The fix wasn't just adding the correct entry — I had to remove 
the wrong one first:
R1(config)#no ip name-server 10.10.10.1
R1(config)#ip name-server 10.10.10.10

If I had just added the correct entry without removing the 
wrong one, R1 would have had two name-servers configured — 
one of which pointed to itself. That would cause inconsistent 
and confusing behaviour depending on which server it tried 
first.

Lesson learned — always read error output slowly. The answer 
is usually already in there. And when correcting a name-server 
entry, remove the wrong one before adding the right one.

---

## What Working DNS Actually Looks Like

Once everything was configured correctly the ping output 
changed noticeably:

R1#ping R2
Translating "R2"...domain server (10.10.10.10)
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.10.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5)

That `Translating "R2"...domain server (10.10.10.10)` line is 
the confirmation — the router contacted the DNS server, got an 
IP back, and used it. Before my fixes that line either showed 
the wrong IP or didn't appear at all.

---

## ARP Behaviour — Something That Surprised Me

After getting DNS working I checked the ARP cache on R1:
```
R1#show arp
Protocol  Address      Age  Hardware Addr    Type  Interface
Internet  10.10.10.1   -    0090.0CD7.0D01   ARPA  FastEthernet0/0
Internet  10.10.10.2   4    0004.9A96.A9A5   ARPA  FastEthernet0/0
Internet  10.10.10.10  2    0090.21C6.D284   ARPA  FastEthernet0/
```
No entry for R3 at 10.10.20.1 — even though I had successfully 
pinged R3 by hostname from R1.

I understood theoretically that ARP is a broadcast protocol 
and broadcasts don't cross router boundaries — but I still 
half-expected to see R3 in there after pinging it. Seeing 
the cache confirmed it physically:

- R1 only has ARP entries for devices on its directly 
  connected network (10.10.10.0/24)
- R3 lives on 10.10.20.0/24 which is across a router
- When R1 pings R3, it ARPs for R2 (the next hop) — 
  not for R3 directly
- So R2's MAC appears in R1's ARP cache, not R3's

R2's ARP cache told the full story — sitting at the boundary 
of both networks it had entries for everything:
```
R2#show arp
Protocol  Address      Age  Hardware Addr    Type  Interface
Internet  10.10.10.1   4    0090.0CD7.0D01   ARPA  FastEthernet0/0
Internet  10.10.10.2   -    0004.9A96.A9A5   ARPA  FastEthernet0/0
Internet  10.10.10.10  1    0090.21C6.D284   ARPA  FastEthernet0/0
Internet  10.10.20.1   4    0030.F2BA.30E7   ARPA  FastEthernet1/0
Internet  10.10.20.2   -    0060.2FCA.ACA0   ARPA  FastEthernet1/0
```
R2 sees both worlds. R1 and R3 each only see their own.

---

## Summary of Mistakes

| Mistake | What Happened | Fix |
|---------|--------------|-----|
| Skipped `ip domain-lookup` | Router didn't attempt DNS at all | Add `ip domain-lookup` before name-server |
| Wrong DNS server IP | Router queried itself instead of DNS server | `no ip name-server [wrong ip]` then add correct one |

---

## Commands That Matter Here:
```
ip domain-lookup              → enables DNS resolution on the router
ip name-server [ip]           → sets which server to send DNS queries to
no ip name-server [ip]        → removes a specific name-server entry
show arp                      → view the ARP cache
ping [hostname]               → tests DNS resolution and connectivity together
```
