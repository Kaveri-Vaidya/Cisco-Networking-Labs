## What this lab is about
Configuring both RIPv2 and EIGRP on a four-router topology, then seeing what happens when two protocols compete over the same networks. The interesting part isn't the config — it's the routing table at the end, where EIGRP's lower administrative distance quietly pushes RIP out for internal routes while RIP still owns the path to the outside world.

---

## Topology

<img width="889" height="656" alt="Screenshot 2026-05-29 181526" src="https://github.com/user-attachments/assets/8978b0fc-4898-43b2-9d3b-f6b1798c4d09" />
Four internal routers (R1–R4), all interconnected over 10.0.x.x/24 subnets. R4 has an uplink to a simulated Service Provider at 203.0.113.0/24. PCs hang off R1 and R3.

---

## Field Notes

### Standing up RIPv2

The config is straightforward — enter the RIP process, declare version 2, kill auto-summary, advertise the classful 10.0.0.0 block:

```
R1(config)#router rip
R1(config-router)#version 2
R1(config-router)#no auto-summary
R1(config-router)#network 10.0.0.0
```

Same commands on every router. The `no auto-summary` matters here — without it, RIPv2 collapses back to classful behaviour and the /24 subnets stop being advertised individually.

After convergence, R1's table shows all remote 10.x.x.x subnets learned via RIP with AD 120:

```
R 10.1.0.0/24 [120/1] via 10.0.0.2, 00:00:00, FastEthernet0/0
R 10.1.1.0/24 [120/2] via 10.0.0.2, 00:00:00, FastEthernet0/0
               [120/2] via 10.0.3.2, 00:00:10, FastEthernet1/1
R 10.1.2.0/24 [120/2] via 10.0.3.2, 00:00:10, FastEthernet1/1
R 10.1.3.0/24 [120/1] via 10.0.3.2, 00:00:24, FastEthernet1/1
```

10.1.1.0/24 shows up via two paths with equal metric — RIP load-balancing at work again, same behaviour as in Lab 16.

### Passive interface toward the Service Provider

R4 connects to the SP at 203.0.113.2. The goal: include that network in RIP so internal routers know how to reach it, but don't leak internal routing updates out to the SP.

```
R4(config)#router rip
R4(config-router)#passive-interface f1/1
R4(config-router)#network 203.0.113.0
```

`passive-interface` stops RIP hellos from going out that interface. The network still gets added to the RIP process so it can be advertised *inward* — just no updates leak outward. Clean separation.

After this, 203.0.113.0/24 shows up on R1 with a hop count of 2:

```
R 203.0.113.0/24 [120/2] via 10.0.3.2, 00:00:12, FastEthernet1/1
```

### Injecting a default route via RIP

R4 gets a static default pointing to the SP:

```
R4(config)#ip route 0.0.0.0 0.0.0.0 203.0.113.2
```

Then that default gets redistributed into RIP so everyone else learns it:

```
R4(config)#router rip
R4(config-router)#default-information originate
```

`default-information originate` is the key command — without it, the static default stays local to R4 and the other routers stay in the dark. After propagation, R1 shows:

```
Gateway of last resort is 10.0.3.2 to network 0.0.0.0

R* 0.0.0.0/0 [120/2] via 10.0.3.2, 00:00:13, FastEthernet1/1
```

The `R*` tag means RIP-learned candidate default. Gateway of last resort is set. All internal routers now have an Internet path.

### Adding EIGRP to the mix

With RIP already converged, EIGRP AS 100 gets configured on every router:

```
R1(config)#router eigrp 100
R1(config-router)#network 10.0.0.0
```

No `no auto-summary` needed here by default in Packet Tracer — the classful 10.0.0.0 advertisement covers all the /24 subnets. EIGRP forms adjacencies fast:

```
R1#sh ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address      Interface   Hold  Uptime    SRTT  RTO   Q   Seq
                              (sec)           (ms)       Cnt Num
0   10.0.0.2     Fa0/0        11   00:00:20   21   126   0   10
1   10.0.3.2     Fa1/1        11   00:00:10   44   264   0   6
```

### Watching AD settle the protocol fight

Both RIP and EIGRP are now advertising the 10.x.x.x networks. AD decides:

- EIGRP: 90
- RIP: 120

Lower wins. The routing table makes it concrete:

```
D 10.1.0.0/24 [90/30720] via 10.0.0.2, 00:06:39, FastEthernet0/0
D 10.1.1.0/24 [90/33280] via 10.0.0.2, 00:06:21, FastEthernet0/0
D 10.1.2.0/24 [90/35840] via 10.0.0.2, 00:06:15, FastEthernet0/0
D 10.1.3.0/24 [90/261120] via 10.0.3.2, 00:06:09, FastEthernet1/1
R 203.0.113.0/24 [120/2] via 10.0.3.2, 00:00:22, FastEthernet1/1
R* 0.0.0.0/0 [120/2] via 10.0.3.2, 00:00:22, FastEthernet1/1
```

All `D` for internal routes. Still `R` for 203.0.113.0/24 and the default — because EIGRP was never told to advertise that network or originate a default. RIP retains exactly what EIGRP isn't doing.

One thing worth noticing: `D 10.1.3.0/24 [90/261120]` has a dramatically higher metric than the others. EIGRP's composite metric (bandwidth + delay) is sensitive to interface types — that higher number reflects different link characteristics on R4's side, not a problem with the config.

---

## Arsenal

```
router rip                          # Enter RIP config mode
version 2                           # Enable RIPv2 (multicast updates, supports VLSM)
no auto-summary                     # Disable classful summarisation
network <classful-network>          # Advertise interfaces in this classful range
passive-interface <int>             # Stop sending RIP updates out this interface
default-information originate       # Inject default route into RIP
router eigrp <AS>                   # Enter EIGRP config mode, specify AS number
network <classful-network>          # Same syntax as RIP for interface matching
show ip route                       # Full routing table — check codes: R vs D
show ip eigrp neighbors             # Verify adjacencies formed (Hold timer, SRTT)
```

---

## Traps I Fell Into

**Forgot `no auto-summary` on RIPv2.** The subnets stopped being individually advertised — R1 was seeing a summarised 10.0.0.0/8 instead of the actual /24s. The routes existed but were useless for longest-prefix matching. Always explicitly disable it on RIPv2.

**Expected EIGRP to take over the default route too.** After configuring EIGRP, I assumed the default route would flip to `D` as well. It didn't — because the default was injected into RIP via `default-information originate` on R4, and EIGRP has no equivalent configured. The `R*` entry staying put was correct behaviour, not a bug.

**Mistook the high EIGRP metric on 10.1.3.0/24 for a problem.** The `[90/261120]` looked off compared to the other EIGRP routes in the 30k–35k range. It's not broken — EIGRP's composite metric reacts to bandwidth and delay, and a slower or different interface type on R4 pushes that number up. The route is still preferred over RIP's AD 120.

---

## Onwards To

OSPF configuration — single-area first, then looking at multi-area and the differences in how LSAs propagate compared to the distance-vector approach here.
