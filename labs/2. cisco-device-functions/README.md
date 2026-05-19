# Cisco Device Functions

## What This Lab Is About
First time actually seeing how switches and routers do their jobs at a hardware level — watching MAC address tables build themselves in real time and seeing how routers decide where to send traffic based on what's directly connected to them.

---

## Topology
Four routers (R1-R4) connected through two switches (SW1 and SW2). All routers preconfigured with IP addresses in the 10.10.10.0/24 network.

                    10.10.10.2                    10.10.10.4
                       R2                              R4
                       |                               |
                      G0/0                            G0/0
                       |                               |
                      F0/2                            F0/4
                       |                               |
                     SW1 ───  F0/24  ───  F0/24  ───  SW2     
                       |                               |                      
                      F0/1                            F0/3       
                       |                                |
                     G0/0                              G0/1
                       |                                |
                  10.10.10.1                        10.10.10.3
                       R1                               R3

---

## Field Notes

### How Switches Actually Learn — MAC Address Tables

Before this lab, I knew switches forwarded traffic but didn't fully appreciate *how* they knew where to send it. Watching it happen live made it click.

When routers started pinging each other, the switches silently watched the traffic and built a table mapping each device's MAC address to the port it came from:

SW1#show mac address-table dynamic
| Vlan | Mac Address | Type | Ports |
|----|------------------------|-----|----|
| 1 | 0090.2b82.ab01 | DYNAMIC | Fa0/1    ← R1 is on port Fa0/1 |
| 1 | 0060.2fb3.9152 | DYNAMIC | Fa0/2    ← R2 is on port Fa0/2 |
| 1 | 0001.9626.8970 | DYNAMIC | Fa0/24   ← R3 is through SW2 |
| 1 | 00d0.9701.02a9 | DYNAMIC | Fa0/24   ← R4 is through SW2 |

R3 and R4 showing up on Fa0/24 makes sense — that's the uplink port connecting SW1 to SW2. SW1 can reach them but only through SW2.

**Cleared the table and immediately checked again:**
SW1#clear mac address-table dynamic
SW1#show mac address-table dynamic

Entries came right back. Real network devices are constantly sending traffic so the table rebuilds almost instantly. The switch will also periodically flush old entries on its own.

### How Routers Actually Learn — Routing Tables

The routing table on R1 before doing anything:
C    10.10.10.0/24    is directly connected, GigabitEthernet0/0
L    10.10.10.1/32    is directly connected, GigabitEthernet0/0

Two routes appeared automatically just from having an IP address configured on an interface:
- **C (Connected)** — the whole network this interface belongs to
- **L (Local)** — the specific IP address of this interface itself

No routes to anywhere else yet — the router only knows what it's directly plugged into.

### Bringing an Interface Online

Added IP 10.10.20.1/24 to GigabitEthernet0/1 on R1 but it stayed administratively down. This is a key difference between routers and switches:

| Device | Interface default state |
|--------|------------------------|
| Router | Administratively down — must manually bring up |
| Switch | Already up by default |

The fix:
R1(config)#interface GigabitEthernet 0/1
R1(config-if)#no shutdown

After bringing it up, two new routes appeared automatically:
C    10.10.20.0/24    is directly connected, GigabitEthernet0/1
L    10.10.20.1/32    is directly connected, GigabitEthernet0/1

The router now knew about both networks and could route between them.

### Adding a Static Route

Told R1 how to reach a network it wasn't directly connected to:
R1(config)#ip route 10.10.30.0 255.255.255.0 10.10.10.2

R1(config)#ip route 10.10.30.0 255.255.255.0 10.10.10.2

This says: *"To reach 10.10.30.0/24, send traffic to 10.10.10.2 and let that router figure out the rest."*

It showed up in the routing table marked with **S** for Static:
S    10.10.30.0/24    [1/0] via 10.10.10.2

---

## Tools of The Expedition
show ip interface brief               → quick status of all interfaces with IPs
show interface gig0/0                 → detailed info including MAC address
show mac address-table dynamic        → see what the switch has learned
clear mac address-table dynamic       → wipe the switch's learned MAC entries
show ip route                         → view the routing table
ip address [ip] [subnet mask]         → assign IP to an interface
no shutdown                           → bring a router interface online
ip route [network] [mask] [next-hop]  → add a static route manually
ping [ip address]                     → test connectivity to a destination

---

## Rough Terrain

- R3 using GigabitEthernet0/1 instead of GigabitEthernet0/0 like everyone else was easy to miss — always verify with `show ip interface brief` rather than assuming.
- Forgetting `no shutdown` after configuring an IP address on a router interface is apparently a classic trap. The IP gets configured successfully but nothing works because the interface is still down.
  Worth double checking every time.

---

## Next Destination
[Lab 12 - The Life of a Packet](../12-life-of-a-packet/README.md)
