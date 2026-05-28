## What this lab is about
This lab is about getting the foundational configuration done 
on routers and switches — the stuff that needs to happen before 
anything else can work. Hostnames, IPs, management access, 
interface descriptions. Then it goes deeper into something I 
found genuinely interesting: what actually happens at the 
physical layer when speed and duplex settings don't agree. 
CDP also made an appearance — Cisco's built-in neighbor 
discovery protocol that I hadn't touched before.

---
<img width="680" height="205" alt="Screenshot 2026-05-28 102424" src="https://github.com/user-attachments/assets/671b3314-2880-4624-9a8d-82a36d82fc17" />

---

## Field Notes

### Initial Configuration — The Baseline

Hostnames first on everything, then IPs on the routers:

```
R1(config)#interface FastEthernet0/0
R1(config-if)#ip address 10.10.10.1 255.255.255.0
R1(config-if)#no shutdown
```
The switch part was new territory. Switches don't get IP 
addresses on physical interfaces — they get them on a virtual 
interface called Vlan1. This is purely for management access 
(SSH, Telnet) — the switch doesn't need an IP to forward 
traffic, that happens at Layer 2 regardless.

SW1(config)#interface vlan1
SW1(config-if)#ip address 10.10.10.10 255.255.255.0
SW1(config-if)#no shutdown

### Default Gateway on a Switch

Since the switch now has a management IP it also needs to know 
how to reach other subnets — otherwise you can only manage it 
from devices on the same network. Switches don't run routing 
so instead of a routing table they just need a single default 
gateway:

SW1(config)#ip default-gateway 10.10.10.2

This is different from how routers handle it — routers use 
routing tables, switches use this single command. Worth 
remembering.

### Interface Descriptions

Simple but genuinely useful in real networks — labeling 
interfaces so anyone looking at the config knows immediately 
what's connected where:
R1(config-if)#description Link to SW1
SW1(config-if)#description Link to R1

Shows up in `show interface` output and makes troubleshooting 
significantly faster when you're staring at a device with 
24+ ports.

### CDP — Cisco Discovery Protocol

CDP runs automatically on Cisco devices and advertises 
information about directly connected neighbors. One command 
reveals everything attached:
SW1#show cdp neighbors
Device ID    Local Intrfce    Holdtme    Capability    Platform    Port ID
R1           Fas 0/1          170        R             C2800       Fas 0/0
R2           Fas 0/2          134        R             C2800       Fas 0/0

SW1 can see both routers, which interfaces they're connected 
to, what platform they are and what port they're using on 
their end. All of this without any manual configuration — 
CDP does it automatically.

**Disabling CDP on a specific interface:**
SW1(config)#interface FastEthernet 0/1
SW1(config-if)#no cdp enable

This stops CDP advertisements going out that specific port — 
R1 can no longer discover SW1. To flush R1's cached CDP 
information immediately:

R1(config)#no cdp run
R1(config)#cdp run

CDP disabled per interface is useful in security contexts — 
you don't want to advertise device information to untrusted 
parts of the network.

### Speed and Duplex — Auto vs Manual

By default interfaces negotiate speed and duplex 
automatically. Verified on the link to R1:
SW1#show interface f0/1
Full-duplex, 100Mb/s

Auto negotiation worked perfectly — both sides agreed on 
100Mbps full duplex without any configuration.

Manually configured the link to R2 to match:
SW1(config-if)#speed 100
SW1(config-if)#duplex full

When manually configuring speed and duplex — both sides 
must match. Configured R2 to match as well:

R2(config-if)#speed 100
R2(config-if)#duplex full

What happens when they don't match is documented separately 
in the troubleshooting logs →
[Speed and Duplex Mismatch](../../troubleshooting-logs/speed-duplex-mismatch.md)

### Reading Interface Status

The `show ip interface brief` output has two status columns 
that mean different things:

| Status | Protocol | Meaning |
|--------|----------|---------|
| up | up | Interface working normally |
| administratively down | down | Manually shut down with `shutdown` command |
| up | down | Physical connection ok but something wrong at layer 2 |
| down | down | Physical layer problem or speed mismatch |

The up/down combination is the interesting one — it means 
the cable is connected and the physical layer looks fine from 
one end, but the two sides can't agree on something at layer 2. 
Speed mismatch produces exactly this on the router side while 
the switch shows down/down. Two devices, same broken link, 
two different views of it.

---

## Arsenal
```
interface vlan1                        → access the switch management interface
ip address [ip] [mask]                 → assign IP to an interface
no shutdown                            → bring interface up
ip default-gateway [ip]                → set default gateway on a switch
description [text]                     → label an interface
show cdp neighbors                     → view directly connected Cisco devices
no cdp enable                          → disable CDP on a specific interface
no cdp run / cdp run                   → flush and restart CDP process
show interface [int]                   → detailed interface info including speed/duplex
show ip interface brief                → quick status of all interfaces
speed [10/100/1000]                    → manually set interface speed
duplex [half/full/auto]                → manually set duplex mode
show version                           → check IOS version and hardware info
```

---

## Traps I Fell Into

- When manually setting speed and duplex, both ends need 
  matching config. Configuring only one side breaks the link 
  in ways that look confusing — full write up in 
  [speed-duplex-mismatch.md](../../troubleshooting-logs/speed-duplex-mismatch.md)

- The `no cdp run` then `cdp run` sequence to flush the CDP 
  cache is not obvious — just disabling CDP on the interface 
  doesn't immediately clear what the neighbor already cached. 
  You have to restart the CDP process on the neighbor to 
  force it to forget.

---

## Onwards To
[Routing Fundamentals](../5. routing-fundamentals/README.md)
