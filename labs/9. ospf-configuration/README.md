## What this lab is about

This lab goes deeper into OSPF than the overview in Lab 17 — not just enabling it and watching routes appear, but understanding Router IDs, cost calculation and why the default reference bandwidth is broken for modern links, manually engineering costs to force load balancing, injecting a default route into OSPF, and then converting the whole topology from single-area to multi-area. DR/BDR election gets its own section at the end on a separate segment with R6–R9.

---

## Topology

<img width="889" height="656" alt="Screenshot 2026-05-29 181526" src="https://github.com/user-attachments/assets/8978b0fc-4898-43b2-9d3b-f6b1798c4d09" />

R1–R5 form the main topology with a Service Provider uplink at R4 (203.0.113.0/24). R6–R9 are on a shared Ethernet segment (172.16.0.0/24) used for the DR/BDR section. All routers have loopback interfaces at 192.168.0.x/32.

---

## Field Notes

### Router IDs — Why Loopbacks Matter

Before enabling OSPF, configured a loopback on every router:

```
R1(config)#interface loopback0
R1(config-if)#ip address 192.168.0.1 255.255.255.255
```

OSPF needs a Router ID to identify itself in the LSDB and to other routers. The selection order: manually configured ID first, then highest loopback IP, then highest physical interface IP. Without a loopback, the Router ID becomes whatever the highest active interface address happens to be at startup — unstable if interfaces go up and down.

With loopbacks configured, OSPF picked them up automatically:

```
R1#sh ip protocols
Router ID 192.168.0.1
```

Every router in the domain now has a stable, predictable identity that matches its loopback. This matters for reading OSPF databases and troubleshooting adjacency issues — you always know which router is which.

### Enabling Single-Area OSPF

```
R1(config)#router ospf 1
R1(config-router)#network 10.0.0.0 0.255.255.255 area 0
R1(config-router)#network 192.168.0.0 0.0.0.255 area 0
```

Two network statements — one for the transit links, one to include the loopbacks. OSPF uses wildcard masks, not subnet masks. The wildcard here is the inverse of the subnet mask: `0.255.255.255` matches anything in the 10.x.x.x space.

Verified adjacencies came up:

```
R1#show ip ospf neighbor
Neighbor ID   Pri   State     Dead Time   Address    Interface
192.168.0.5   1     FULL/BDR  00:00:31    10.0.3.2   FastEthernet1/1
192.168.0.2   1     FULL/DR   00:00:39    10.0.0.2   FastEthernet0/0
```

The `FULL` state means the LSDB is synchronised with that neighbor. `DR` and `BDR` show up here too — more on that later.

After convergence, R1's table had all 10.x.x.x networks and all five loopbacks via OSPF. Two routes to 10.1.1.0/24 with equal cost — OSPF load balancing on equal-cost paths just like RIP, but for different reasons.

### The Reference Bandwidth Problem

OSPF calculates cost as: `reference bandwidth / interface bandwidth`. The default reference bandwidth is 100 Mbps. That means:

- 100 Mbps FastEthernet → cost 1
- 10 Mbps → cost 10
- 1 Gbps → cost 1 (same as 100 Mbps — OSPF can't differentiate)
- 10 Gbps → cost 1 (still the same)

Anything faster than the reference bandwidth hits the floor of 1. In a modern network with Gigabit and 10G interfaces, every high-speed link looks identical to OSPF. It can't make intelligent path decisions.

Fixed it by setting the reference bandwidth to 100 Gbps across all routers:

```
R1(config)#router ospf 1
R1(config-router)#auto-cost reference-bandwidth 100000
```

Now cost = 100000 / interface_bandwidth_in_Mbps. A 100 Mbps FastEthernet link has cost 1000. A 1 Gbps link would be 100. A 10 Gbps link would be 10. The protocol can now distinguish between them.

Verified on an interface:

```
R1#show ip ospf interface FastEthernet 0/0
Cost: 1000
```

The metric values in the routing table jumped from single digits to thousands — same relative relationships, just with headroom for faster links to matter:

```
O 10.1.0.0/24 [110/2000] via 10.0.0.2   ← was [110/2]
O 10.1.2.0/24 [110/3000] via 10.0.3.2   ← was [110/3]
```

This must be configured consistently on every router in the domain. If one router uses a different reference bandwidth, its cost calculations will be inconsistent with everyone else and traffic will follow unexpected paths.

### Engineering Load Balancing With Manual Costs

After the reference bandwidth change, R1 had two possible paths to 10.1.2.0/24:

- Via R5 → R4: cost 1000 (R1→R5) + 1000 (R5→R4) + 1000 (R4's interface) = **3000**
- Via R2 → R3 → R4: cost 1000 + 1000 + 1000 + 1000 = **4000**

OSPF chose the R5 path. To force load balancing across both, needed to make them equal cost. The approach: manually set the R1→R5 and R5→R4 links to cost 1500 each.

```
R1(config)#int f1/1
R1(config-if)#ip ospf cost 1500

R5(config)#int f0/0
R5(config-if)#ip ospf cost 1500
R5(config)#int f0/1
R5(config-if)#ip ospf cost 1500

R4(config)#int f1/0
R4(config-if)#ip ospf cost 1500
```

Now R5 path = 1500 + 1500 + 1000 = 4000. Equal to the R2 path. Both show up in the routing table:

```
O 10.1.2.0/24 [110/4000] via 10.0.3.2, FastEthernet1/1
               [110/4000] via 10.0.0.2, FastEthernet0/0
```

This is the way to tune OSPF traffic engineering — not by touching the protocol itself, but by adjusting the cost values it uses for its calculations.

### Default Route Injection Into OSPF

Same pattern as the RIP and EIGRP labs. R4 connects to the SP — passive interface facing the SP, add that network to OSPF, then configure a default static and redistribute it:

```
R4(config)#router ospf 1
R4(config-router)#passive-interface f1/1
R4(config-router)#network 203.0.113.0 0.0.0.255 area 0

R4(config)#ip route 0.0.0.0 0.0.0.0 203.0.113.2
R4(config)#router ospf 1
R4(config-router)#default-information originate
```

The injected default shows up on R1 with a specific code:

```
O*E2 0.0.0.0/0 [110/1] via 10.0.3.2, FastEthernet1/1
               [110/1] via 10.0.0.2, FastEthernet0/0
```

`O*E2` means OSPF external type 2 — a route redistributed into OSPF from outside the domain. The metric stays fixed at 1 regardless of how many hops it travels within OSPF. That's the E2 behaviour: cost is the external cost only, not accumulated internal cost. Load balanced via both R2 and R5 because the cost is the same on both paths.

### Multi-Area OSPF — Splitting the Topology

Single area works fine for small topologies. As the LSDB grows, every router stores and processes the complete topology — routers in one corner of the network are flooded with LSAs about links they'll never use. Multi-area solves this by confining LSA flooding to an area, with Area Border Routers summarising between areas.

The target layout:

- **Area 0 (backbone)**: R3, R4
- **Area 1**: R1
- **ABRs**: R2 (one interface in Area 0, one in Area 1), R5 (same)

R1's network statements needed to point to Area 1 instead of Area 0:

```
R1(config)#router ospf 1
R1(config-router)#network 10.0.0.0 0.255.255.255 area 1
R1(config-router)#network 192.168.0.0 0.0.0.255 area 1
```

R2 needed more granular statements — it had a single `10.0.0.0/8` wildcard covering everything, which can't split interfaces between areas:

```
R2(config)#router ospf 1
R2(config-router)#no network 10.0.0.0 0.255.255.255 area 0
R2(config-router)#network 10.1.0.0 0.0.0.255 area 0   ← backbone-facing interface
R2(config-router)#network 10.0.0.0 0.0.0.255 area 1   ← R1-facing interface
```

After reloading, verified with `show ip ospf interface` on R2:

```
FastEthernet0/1   Area 0   ← toward backbone
FastEthernet0/0   Area 1   ← toward R1
```

R1's routing table now showed `O IA` instead of `O` for routes beyond its own area:

```
O IA 10.1.0.0/24  [110/2000] via 10.0.0.2
O IA 10.1.1.0/24  [110/3000] via 10.0.0.2
```

`IA` = inter-area. R1 still has full reachability but the routes now come through the ABR summaries rather than from full LSA flooding across the topology.

### Inter-Area Summarisation — The Payoff of Multi-Area

After moving to multi-area, R1 still had the same number of routing table entries. OSPF doesn't auto-summarise — it has to be explicitly configured on the ABRs:

```
R2(config)#router ospf 1
R2(config-router)#area 0 range 10.1.0.0 255.255.0.0
R2(config-router)#area 1 range 10.0.0.0 255.255.0.0

R5(config)#router ospf 1
R5(config-router)#area 0 range 10.1.0.0 255.255.0.0
R5(config-router)#area 1 range 10.0.0.0 255.255.0.0
```

After this, R1's table went from four separate 10.1.x.x/24 entries to one summary:

```
O IA 10.1.0.0/16 [110/2000] via 10.0.0.2, FastEthernet0/0
```

Four routes collapsed into one. In a large deployment this is significant — routers don't need to know about every individual subnet in every other area, just the summarised prefix. The LSDB shrinks, reconvergence is faster, and memory usage drops.

### DR and BDR — Designated Router Election

On the R6–R9 segment, four routers share a single Ethernet broadcast domain (172.16.0.0/24). Without a DR, every OSPF router would need a full adjacency with every other router — on a segment with n routers that's n*(n-1)/2 adjacencies. With 4 routers that's 6 full adjacencies. With 10 it's 45. Doesn't scale.

OSPF solves this by electing a Designated Router. All other routers form a full adjacency only with the DR (and BDR as backup). Non-DR routers are in `2WAY/DROTHER` state with each other — they know each other exist but don't exchange full LSDBs.

Election rules (in priority order):
1. Highest OSPF priority wins (default is 1 for all)
2. Tie-break: highest Router ID

With all four routers at default priority 1, R9 won (Router ID 192.168.0.9 — highest loopback). R8 became BDR. R6 and R7 were DROTHERs.

```
R6#show ip ospf interface FastEthernet 0/0
State DROTHER, Priority 1
Designated Router (ID) 192.168.0.9
Backup Designated Router (ID) 192.168.0.8
```

To force R6 to become DR without changing any IP addresses, bumped its OSPF priority and reset the process:

```
R6(config)#interface FastEthernet0/0
R6(config-if)#ip ospf priority 100
R6(config-if)#end
R6#clear ip ospf process
```

Priority 100 beats the default 1 on every other router. After re-election:

```
R6#show ip ospf interface FastEthernet 0/0
State DR, Priority 100
Designated Router (ID) 192.168.0.6
```

Important caveat: OSPF DR elections are non-preemptive. If R6 had set priority 100 while R9 was already DR, R9 would have remained DR until the process was cleared or it went down. `clear ip ospf process` triggers a re-election.

---

## Arsenal

```
interface loopback0                             # Create loopback
ip address 192.168.0.x 255.255.255.255          # /32 host address on loopback

router ospf 1                                   # Enable OSPF process
network [net] [wildcard] area [n]               # Enable OSPF on matching interfaces
auto-cost reference-bandwidth 100000            # Set ref BW to 100 Gbps (Mbps unit)
default-information originate                   # Inject default static into OSPF
passive-interface [int]                         # Stop OSPF hellos, still advertise

ip ospf cost [value]                            # Override cost on an interface
ip ospf priority [value]                        # Override DR priority on an interface
clear ip ospf process                           # Reset OSPF, trigger re-election

area [n] range [net] [mask]                     # Summarise on ABR between areas
no network [net] [wildcard] area [n]            # Remove a network statement

show ip protocols                               # Verify Router ID, networks, AD
show ip ospf neighbor                           # Adjacency states, DR/BDR roles
show ip ospf interface [int]                    # Cost, area, DR/BDR, priority
show ip ospf database                           # Full LSDB — Router, Net, Summary LSAs
show ip route                                   # O = intra-area, O IA = inter-area, O*E2 = external
```

---

## Traps I Fell Into

**Forgot to set reference bandwidth on every router.** Set it on R1 and R2 but not R3. R3 was still calculating costs with the 100 Mbps default so its view of the topology was completely different. Routes that looked symmetric from R1 were asymmetric from R3's perspective. OSPF cost tuning only works when every router in the domain agrees on the reference — it has to be a consistent, global change.

**Tried to split R2's interfaces between areas using the wildcard `0.255.255.255`.** That matches all 10.x.x.x addresses and puts every interface matching that range into the same area. Can't use one broad statement to put some interfaces in Area 0 and others in Area 1. Had to `no` the broad statement and replace it with two specific ones — one /24 wildcard per interface.

**Expected R1's routing table to shrink after moving to multi-area.** It didn't — still had four separate 10.1.x.x/24 entries. Multi-area alone doesn't summarise anything. Had to explicitly configure `area range` on both ABRs. Multi-area creates the structure; summarisation is a separate step that takes advantage of it.

**The DR election didn't change when I set priority without clearing the process.** OSPF won't preempt a sitting DR just because a higher-priority router joins. The new priority is noted but the existing DR keeps its role until it goes down or the process is reset. Had to run `clear ip ospf process` to trigger a fresh election.

---

## Onwards To

EIGRP configuration in depth — applying the same cost-engineering mindset to a composite metric protocol.
