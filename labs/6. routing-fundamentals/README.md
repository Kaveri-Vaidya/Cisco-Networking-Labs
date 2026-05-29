## What this lab is about
This lab is about getting routing to actually work across a 
multi-router topology from scratch — configuring connected, 
local, static, summary and default routes and watching how 
the routing table changes at each step. The longest prefix 
match section was the most interesting part — seeing two 
routes exist for the same destination and watching the 
router automatically pick the more specific one without 
any extra configuration.

---

## Topology

<img width="889" height="656" alt="Screenshot 2026-05-29 181526" src="https://github.com/user-attachments/assets/24fed82d-53b3-40b7-b185-94bca5f0bc3d" />

Networks used:
```
- R1 side: 10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24
- R4 side: 10.1.1.0/24, 10.1.2.0/24, 10.1.3.0/24
- Internet link: 203.0.113.0/24
```
---

## Field Notes

### Connected and Local Routes — What Appears Automatically

Configured IPs on all four of R1's interfaces and 
immediately checked the routing table without adding 
anything manually:
```
R1#show ip route
C  10.0.1.0/24   is directly connected, FastEthernet0/1
L  10.0.1.1/32   is directly connected, FastEthernet0/1
C  10.0.2.0/24   is directly connected, FastEthernet1/0
L  10.0.2.1/32   is directly connected, FastEthernet1/0
```
Only two networks appeared — not four. The interfaces 
connected to R2 and R5 were missing because router 
interfaces are administratively down by default. Both 
sides of a link must be up for a route to appear. The 
interfaces connected to SW1 and SW2 showed up immediately 
because switch ports are already up by default.

Two route types appeared automatically for each active 
interface:
- **C (Connected)** — the whole subnet this interface 
  belongs to
- **L (Local)** — just this router's specific IP address 
  as a /32 host route

The L route is what the router uses to identify traffic 
destined for itself rather than traffic it needs to 
forward onwards.

### PC1 to PC2 — Works Without Static Routes

Pinged PC2 (10.0.2.10) from PC1 (10.0.1.10) before 
configuring any static routes — it worked. Both PCs 
are on networks directly connected to R1 so no static 
routes needed. R1 already knows about both networks 
from its connected routes.

Traceroute confirmed the path:
```
C:>tracert 10.0.2.10
1   10.0.1.1    ← R1
2   10.0.2.10   ← PC2
```
### PC1 to PC3 — Fails Without Static Routes

Tried pinging PC3 (10.1.2.10) — failed immediately 
with "Destination host unreachable" coming back from 
R1. R1 had no route to the 10.1.x.x networks at all 
so it rejected the traffic at the first hop.

This is the fundamental limitation of connected routes 
— a router only knows about what's directly attached 
to it. Everything else needs to be told explicitly.

### Static Routes — Connecting the Full Topology

Configured IPs on R2, R3 and R4, then added static 
routes on every router to reach every subnet:

On R1 (pointing everything toward R2 as the next hop):
```
R1(config)#ip route 10.1.0.0 255.255.255.0 10.0.0.2
R1(config)#ip route 10.1.1.0 255.255.255.0 10.0.0.2
R1(config)#ip route 10.1.2.0 255.255.255.0 10.0.0.2
R1(config)#ip route 10.1.3.0 255.255.255.0 10.0.0.2
```
Every router needed routes in both directions — 
outbound to the destination and return routes back 
to the source. Missing a return route is a common 
mistake that causes one-way connectivity, which 
shows up as pings timing out even though traceroute 
shows the traffic reaching the destination.

Verified the full path from PC1 to PC3:
```
C:>tracert 10.1.2.10
1   10.0.1.1   ← R1
2   10.0.0.2   ← R2
3   10.1.0.1   ← R3
4   10.1.1.1   ← R4
5   10.1.2.10  ← PC3
```
### Summary Routes — Four Commands Becomes One
```
Removed all four static routes from R1:
R1(config)#no ip route 10.1.0.0 255.255.255.0 10.0.0.2
R1(config)#no ip route 10.1.1.0 255.255.255.0 10.0.0.2
R1(config)#no ip route 10.1.2.0 255.255.255.0 10.0.0.2
R1(config)#no ip route 10.1.3.0 255.255.255.0 10.0.0.2
```
Connectivity to PC3 dropped immediately. Then restored 
all of it with a single summary route:
```
R1(config)#ip route 10.1.0.0 255.255.0.0 10.0.0.2
```
One /16 route covered all four /24 subnets on the 
R4 side. The routing table went from four S entries 
to one:
```
S  10.1.0.0/16 [1/0] via 10.0.0.2
```
Summary routes keep routing tables smaller and easier 
to manage. In large networks with hundreds of subnets 
the difference between summarising and not summarising 
is the difference between a readable routing table and 
an unmanageable one.

### Longest Prefix Match — The Most Interesting Part

Added R5 to the topology with two interfaces:
- F0/0: 10.1.3.2 (connecting to R4 side)
- F0/1: 10.0.3.2 (connecting directly to R1)

Without any additional routes, PC1 trying to reach 
R5's F0/0 interface at 10.1.3.2 sent traffic the 
long way — R1 > R2 > R3 > R4 > R5 — because the 
summary route pointed all 10.1.x.x traffic toward R2. 
R5 had no return route so pings failed.

Added a summary route on R5 back to R1's networks:
```
R5(config)#ip route 10.0.0.0 255.255.0.0 10.0.3.1
```
Traffic now reached R5 via R1 > R2 > R3 > R4 > R5 
but the return path was R5 > R1 directly — 
asymmetric routing. Both paths worked but traffic 
was taking different routes each direction.

To force traffic to R5's F0/0 to take the direct 
path in both directions, added a more specific route 
on R1:
```
R1(config)#ip route 10.1.3.0 255.255.255.0 10.0.3.2
```
Now R1's routing table had two routes that matched 
traffic to 10.1.3.2:
```
S  10.1.0.0/16  [1/0] via 10.0.0.2   ← summary route
S  10.1.3.0/24  [1/0] via 10.0.3.2   ← specific route
```
The /24 is more specific than the /16 so the router 
automatically used it — longest prefix match. No 
additional configuration needed, no priority settings, 
no tie-breaking rules. The router just picks the most 
specific match every time.

Verified with traceroute from PC1 to R5's F0/0:
```
C:>tracert 10.1.3.2
1   10.0.1.1    ← R1
2   10.1.3.2    ← R5 direct
```
Two hops. Previously five.

### Default Route — The Route of Last Resort

Configured the internet-facing interface on R4:
```
R4(config)#int f1/1
R4(config-if)#ip add 203.0.113.1 255.255.255.0
R4(config-if)#no shut
```
Then added default routes on every router pointing 
toward R4's internet connection:
```
R1(config)#ip route 0.0.0.0 0.0.0.0 10.0.0.2
R2(config)#ip route 0.0.0.0 0.0.0.0 10.1.0.1
R3(config)#ip route 0.0.0.0 0.0.0.0 10.1.1.1
R4(config)#ip route 0.0.0.0 0.0.0.0 203.0.113.2
R5(config)#ip route 0.0.0.0 0.0.0.0 10.1.3.1
```
`0.0.0.0 0.0.0.0` matches everything that doesn't 
match a more specific route — it's the catch-all. 
Every packet with an unknown destination gets sent 
toward R4 which passes it to the ISP.

R4's routing table showed this clearly:
```
Gateway of last resort is 203.0.113.2 to network 0.0.0.0
S*  0.0.0.0/0 [1/0] via 203.0.113.2
```
The `*` marks it as the candidate default route.

### Load Balancing — Two Default Routes on R1

Added a second default route on R1 pointing toward R5:
```
R1(config)#ip route 0.0.0.0 0.0.0.0 10.0.3.2
```
R1's routing table showed both paths with equal cost:
```
S*  0.0.0.0/0 [1/0] via 10.0.0.2
[1/0] via 10.0.3.2
```
Two equal-cost paths to the same destination — the 
router automatically load balances traffic across 
both. No configuration needed beyond adding the 
second route. This is built into how IOS handles 
equal-cost routes.

---

## Arsenal
```
show ip route                              → view full routing table
ip route [network] [mask] [next-hop]       → add a static route
no ip route [network] [mask] [next-hop]    → remove a static route
ip route 0.0.0.0 0.0.0.0 [next-hop]       → add a default route
tracert [ip]                               → trace path from a PC
traceroute [ip]                            → trace path from a router
ping [ip]                                  → test connectivity
```
---

## Traps I Fell Into

- When static routes were first configured, connectivity 
  worked in one direction but not the other. The return 
  routes were missing on one of the routers. Ping timing 
  out while traceroute shows traffic reaching the 
  destination is the tell for a missing return route — 
  the traffic gets there but can't find its way back.

- The asymmetric routing with R5 was unexpected. Traffic 
  going to R5 took five hops via R2, R3, R4 but the 
  return came straight back via R1 in one hop. Both 
  paths were technically correct — routers make 
  independent decisions based on their own routing 
  table and don't coordinate with each other. Adding 
  the more specific /24 route on R1 fixed the 
  forward path to also use the direct link.

---

## Onwards To
[Dynamic Routing Protocols](../dynamic-routing-protocols/README.md)
