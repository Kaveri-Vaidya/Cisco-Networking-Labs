## What this lab is about
This lab is about understanding what's actually happening under
the hood when routers talk to each other — not just configuring
a protocol and seeing routes appear, but watching the updates
fly, understanding why one protocol wins over another, seeing
a topology reconverge in real time when a link goes down, and
getting hands on with concepts like administrative distance,
metrics, passive interfaces and floating static routes. Three
IGPs in one lab — RIP, OSPF and EIGRP — each replacing the
previous one and behaving differently in ways that make the
theory click.

---

## Topology

<img width="889" height="656" alt="Screenshot 2026-05-29 181526" src="https://github.com/user-attachments/assets/8978b0fc-4898-43b2-9d3b-f6b1798c4d09" />

---

## Field Notes

### RIPv1 — Broadcast Updates

Enabled basic RIP on every router:
```
router rip
network 10.0.0.0
no auto-summary
```
Turned on debug to watch what was actually happening:
```
R1#debug ip rip
```
RIPv1 sends updates to the broadcast address:
```
RIP: sending v1 update to 255.255.255.255 via FastEthernet0/0
```
Every device on the subnet has to process these packets —
not just routers. Hosts, printers, everything. That's
wasteful and a security concern — any device on the network
can see what routes are being advertised.

### RIPv2 — Multicast Updates

Added version 2 on every router:
```
router rip
version 2
```
Debug output changed immediately:
```
RIP: sending v2 update to 224.0.0.9 via FastEthernet1/0
```
224.0.0.9 is the RIPv2 multicast address. Only devices
that have joined that multicast group — meaning RIPv2
routers — process these packets beyond layer 3. Everything
else ignores them. Cleaner and more secure than broadcast.

### Reading the RIP Routing Table

After RIPv2 converged, R1's routing table showed:
```
R  10.1.0.0/24  [120/1] via 10.0.0.2   ← 1 hop away via R2
R  10.1.1.0/24  [120/2] via 10.0.0.2   ← 2 hops via R2
[120/2] via 10.0.3.2   ← 2 hops via R5
R  10.1.2.0/24  [120/2] via 10.0.3.2
R  10.1.3.0/24  [120/1] via 10.0.3.2   ← 1 hop away via R5
```
The `[120/1]` format means `[AD/metric]`:
- 120 is RIP's administrative distance
- The number after `/` is the hop count metric

Two routes to 10.1.1.0/24 with equal hop count — RIP
installs both and load balances between them automatically.
RIP has no awareness of bandwidth — a 1Gbps link and a
56Kbps link both count as one hop.

### OSPF Replaces RIP — Administrative Distance

Enabled OSPF on every router while RIP was still running:
```
router ospf 1
network 10.0.0.0 0.255.255.255 area 0
```
After OSPF converged, checked the routing table — all R
entries were gone, replaced by O entries. RIP was still
running and still sending updates but its routes weren't
being used.

The reason is administrative distance:

| Protocol | AD |
|----------|----|
| Connected | 0 |
| Static | 1 |
| EIGRP | 90 |
| OSPF | 110 |
| RIP | 120 |

Lower AD wins. OSPF at 110 beats RIP at 120 so the router
trusts OSPF routes and ignores the RIP ones entirely. Both
protocols were running simultaneously but only one was
actually being used.

### OSPF Uses Bandwidth — Not Just Hop Count

With OSPF running, only one route appeared to 10.1.1.0/24
even though two equal-hop paths existed. The reason was
R5's interfaces had been configured with a bandwidth of
10Mbps:
```
R5#show run | section interface
interface FastEthernet0/0
bandwidth 10000
```
OSPF calculates cost based on bandwidth — higher bandwidth
means lower cost means preferred path. The top path via
R2 used 100Mbps FastEthernet interfaces throughout so its
cost was far lower than the path via R5 at 10Mbps. OSPF
picked the better path automatically.

RIP would have seen both paths as equal — 2 hops each —
and load balanced. OSPF was smarter about which path was
actually faster.

### OSPF Reconvergence — Live Failover

Shut down FastEthernet 0/0 on R2 to simulate a link
failure:
```
R2(config-if)#shutdown
```
Console on R1 immediately showed:
```
%OSPF-5-ADJCHG: Process 1,
Nbr 10.0.3.1 on FastEthernet0/0 from FULL to DOWN,
Neighbor Down: Interface down or detached
````
OSPF detected the adjacency loss and reconverged. Checked
R1's routing table — all routes that previously went via
R2 now went via R5 instead, with higher cost metrics
reflecting the slower path:
```
O  10.1.0.0/24  [110/22] via 10.0.3.2   ← previously [110/2] via R2
O  10.1.1.0/24  [110/21] via 10.0.3.2
O  10.1.2.0/24  [110/21] via 10.0.3.2
O  10.1.3.0/24  [110/20] via 10.0.3.2
```
No manual intervention. No static route changes. The
network adapted on its own. This is the whole point of
dynamic routing.

### RIP vs OSPF Database — Distance Vector vs Link State

Compared the two databases side by side. RIP's database:
```
R1#show ip rip database
10.1.1.0/24
[2] via 10.0.0.2, FastEthernet0/0
[2] via 10.0.3.2, FastEthernet1/1
```
RIP only knows what its directly connected neighbors told
it — a list of networks and distances. It has no idea
what the rest of the topology looks like.

OSPF's database:
```
R1#show ip ospf database
Router Link States (Area 0)
Link ID     ADV Router    Age    Seq#
10.1.1.2    10.1.1.2      563    0x80000004
10.0.3.1    10.0.3.1      218    0x80000007
10.1.3.2    10.1.3.2      558    0x80000004
```
OSPF has an entry for every router in the area — it knows
the state of every link on every device. This is why it
reconverges faster and makes smarter path decisions. It's
working from a complete map of the network rather than
just directions from its neighbors.

### EIGRP Replaces OSPF — Lower AD Wins Again

Removed OSPF and enabled EIGRP:
```
no router ospf 1
router eigrp 100
no auto-summary
network 10.0.0.0 0.255.255.255
```
RIP routes replaced OSPF routes when OSPF was removed,
then EIGRP with AD 90 beat RIP's AD 120 and took over.
The routing table shifted from R entries to D entries.

EIGRP's metric is composite — bandwidth and delay combined
— giving it a large number that looks intimidating but
captures more about the path than a simple hop count:
```
D  10.1.0.0/24  [90/30720]  via 10.0.0.2
D  10.1.3.0/24  [90/261120] via 10.0.3.2  ← much higher cost via R5
```
The 261120 vs 30720 difference reflects R5's 10Mbps
interfaces. EIGRP picked this up just like OSPF did.

### Floating Static Routes — Backup Path With a Twist

Disabled RIP and EIGRP on R5 to simulate a router not
yet running a routing protocol, then needed to ensure
connectivity survived if the R1-R2 link went down.

Added floating static routes — static routes with a
manually set AD higher than EIGRP's 90:
```
R1(config)#ip route 10.1.0.0 255.255.0.0 10.0.3.2 95
```
The `95` at the end is the administrative distance. Because
95 is higher than EIGRP's 90, this route stays invisible
in the routing table as long as EIGRP routes exist — it
floats below the surface. The moment EIGRP loses the path,
the floating static emerges and takes over.

Verified with R1's routing table — both routes visible but
the EIGRP /24 routes were being used, the static /16 was
present but not selected:
```
S  10.1.0.0/16   [95/0] via 10.0.3.2     ← floating, standing by
D  10.1.0.0/24   [90/30720] via 10.0.0.2 ← EIGRP in use
D  10.1.1.0/24   [90/33280] via 10.0.0.2
```
Shut down R2's interface — EIGRP routes disappeared,
floating static stepped in:
```
S  10.1.0.0/16   [95/0] via 10.0.3.2     ← now the active route
```
PC1 to PC3 still worked. Traffic now went via R5 instead
of R2. Brought R2 back up — EIGRP routes came back,
floating static retreated again.

The AD of 95 on R5's routes matters even though R5 wasn't
running EIGRP yet — it was set to 95 to prevent those
routes from becoming preferred when EIGRP gets enabled
on R5 in the future.

### Loopback Interfaces — Always Up

Configured a loopback on every router:
```
R1(config)#interface loopback0
R1(config-if)#ip address 192.168.0.1 255.255.255.255
```
Checked if PC1 could reach them — couldn't. The 192.168.x.x
range wasn't included in EIGRP's network statement so the
loopbacks weren't being advertised. The routing table only
showed each router's own loopback as a local connected
route.

Added the 192.168.0.0 range to EIGRP:
```
R1(config-router)#network 192.168.0.0 0.0.0.255
```
All loopback addresses appeared in R1's routing table
immediately and PC1 could ping R5's loopback at
192.168.0.5.

Loopbacks are configured as /32 host routes — there's only
one address and it never needs a subnet. They're used for
router management, OSPF router IDs, BGP peering and
anywhere you need a stable address that never goes down
even if physical interfaces fail.

### Passive Interfaces — Stop Sending, Keep Receiving

Configured R1's loopback and its link to R5 as passive
interfaces:
```
R1(config-router)#passive-interface loopback0
R1(config-router)#passive-interface fastethernet1/1
```
The moment FastEthernet1/1 became passive, the EIGRP
adjacency with R5 dropped:
```
%DUAL-5-NBRCHANGE: IP-EIGRP 100: Neighbor 10.0.3.2
(FastEthernet1/1) is down: holding time expired
```
R5's routing table shifted — all routes that previously
went via R1 directly now went via R4 instead, with higher
metrics reflecting the longer path.

A passive interface still advertises the connected network
to other EIGRP neighbors — it just stops sending Hello
packets on that interface so no adjacency can form there.
Used on interfaces connected to end devices where a routing
protocol neighborship would be pointless and potentially
a security risk.

---

## Administrative Distance Reference

| Route Source | AD |
|-------------|-----|
| Connected | 0 |
| Static | 1 |
| EIGRP | 90 |
| OSPF | 110 |
| RIP | 120 |
| Unknown/Untrusted | 255 |

Lower is more trusted. When two protocols advertise a
route to the same destination, the one with lower AD wins.

---

## Arsenal
```
router rip                              → enable RIP process
version 2                               → upgrade to RIPv2
network [classful network]              → enable RIP on matching interfaces
no auto-summary                         → disable automatic summarisation
debug ip rip                            → watch RIP updates in real time
undebug all                             → turn off all debugging
show ip rip database                    → view RIP's route database
router ospf 1                           → enable OSPF process
network [network] [wildcard] area [n]   → enable OSPF on matching interfaces
show ip ospf database                   → view OSPF's link state database
no router ospf 1                        → remove OSPF entirely
router eigrp 100                        → enable EIGRP AS 100
network [network] [wildcard]            → enable EIGRP on matching interfaces
no auto-summary                         → disable automatic summarisation
show ip eigrp neighbors                 → verify EIGRP adjacencies
passive-interface [interface]           → stop EIGRP hellos on an interface
no router eigrp 100                     → remove EIGRP entirely
ip route [net] [mask] [next-hop] [AD]   → floating static route with custom AD
interface loopback0                     → create a loopback interface
show ip route                           → view routing table
```
---

## Traps I Fell Into

- When OSPF was enabled while RIP was still running,
  I initially expected to see both sets of routes in
  the table. Only OSPF routes appeared because AD
  decided the winner silently — RIP was still running
  and advertising, the router just wasn't using its
  routes. Both protocols can coexist but only the lower
  AD wins.

- The floating static AD of 95 needs to be set even on
  R5 where EIGRP wasn't running yet. Easy to think it
  doesn't matter now — but when EIGRP gets enabled on
  R5 in the future, those routes would suddenly become
  preferred over EIGRP if the AD wasn't set correctly.
  Configure for the future state, not just the current one.

- Passive interface stops Hello packets on that interface
  which means no adjacency can form — but the network
  is still advertised to other EIGRP neighbors. The
  loopback becoming passive had no effect on routing
  tables. The FastEthernet1/1 to R5 becoming passive
  dropped the adjacency immediately and R5 had to find
  alternate paths.
  
---

## Onwards To
Configuration in depth
