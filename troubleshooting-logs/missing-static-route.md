## The Situation
While working with a multi-router topology using static
routes, PC1 was not able to connect to PC3. The
network had been working for the dynamic routing protocols so something was either
misconfigured or missing from the start. This was a
pure routing table investigation — no interface issues,
no physical problems, just a gap in the routing logic.

---

## Topology at the Time
<img width="889" height="656" alt="Screenshot 2026-05-29 181526" src="https://github.com/user-attachments/assets/8978b0fc-4898-43b2-9d3b-f6b1798c4d09" />

Static routes were configured across all routers.
No dynamic routing protocol was running.

---

## Step 1 — Confirm the Failure
```
C:>ping 10.1.2.10
Request timed out.
Request timed out.
Reply from 10.1.0.1: Destination host unreachable.
Reply from 10.1.0.1: Destination host unreachable.
```
Two things in this output worth reading carefully:
- The first two timeouts mean the router had no idea
  where to send the packet initially
- The last two came back from 10.1.0.1 — that's R3
  sending back an unreachable message

R3 is telling us it received the packet but has nowhere
to forward it. The trail goes cold at R3.

---

## Step 2 — Traceroute to Pinpoint the Hop
C:>tracert 10.1.2.10
1   10.0.1.1   ← R1, fine
2   10.0.0.2   ← R2, fine
3   10.1.0.1   ← R3, fine
4   10.1.0.1   ← R3 again — traffic is bouncing back
5   * 0 ms
Control-C

Traffic reaches R3 successfully but hop 4 shows R3's
address again — the packet is looping back to R3 rather
than continuing forward to R4. R3 is either sending
traffic back toward R2 or simply has no route to
10.1.2.0/24 and is returning an unreachable.

This narrows it to R3. Something is wrong with how R3
handles traffic destined for 10.1.2.0/24.

---

## Step 3 — Check R3's Routing Table
```
R3#show ip route
S  10.0.0.0/24  [1/0] via 10.1.0.2
S  10.0.1.0/24  [1/0] via 10.1.0.2
S  10.0.2.0/24  [1/0] via 10.1.0.2
S  10.0.3.0/24  [1/0] via 10.1.0.2
C  10.1.0.0/24  is directly connected, FastEthernet0/1
L  10.1.0.1/32  is directly connected, FastEthernet0/1
C  10.1.1.0/24  is directly connected, FastEthernet0/0
L  10.1.1.2/32  is directly connected, FastEthernet0/0
S  10.1.3.0/24  [1/0] via 10.1.1.1
```
Found it. R3 has static routes to every network in
the topology except 10.1.2.0/24 — the exact network
PC3 lives on. Every other subnet accounted for, that
one simply missing.

R3 has a route to 10.1.3.0/24 via R4 but nobody
added the route to 10.1.2.0/24. When traffic for
PC3 arrived at R3 it had nowhere to go and sent back
an unreachable.

---

## The Fix
```
R3(config)#ip route 10.1.2.0 255.255.255.0 10.1.1.1
```
Sent traffic toward R4 (10.1.1.1) which is directly
connected to the 10.1.2.0/24 network where PC3 lives.

Verified from PC1:
```
C:>ping 10.1.2.10
Request timed out.     ← ARP cache updating
Request timed out.     ← ARP cache updating
Reply from 10.1.2.10: bytes=32 time<1ms TTL=124
Reply from 10.1.2.10: bytes=32 time=1ms TTL=124
```
The first two timeouts are normal — ARP needs to
resolve before traffic can flow. The last two replies
confirm full restoration.

---

## What Made This Click

The traceroute output showing R3's IP address twice
in a row was the key signal. When a router appears
at the same hop number repeatedly it means traffic
is being sent back toward the source rather than
forwarding toward the destination — a classic sign
of either a missing route or a routing loop.

Checking `show ip route` on R3 immediately confirmed
the missing entry. In a static routing environment
this kind of omission is easy to make — one subnet
forgotten out of many when configuring routes manually.
This is one of the core reasons dynamic routing
protocols exist — they discover and advertise routes
automatically so a single missing entry doesn't bring
down connectivity silently.

---

## The Methodology That Worked

Start from the source, work toward the destination,
stop at the first hop that can't forward properly:

1. Ping to confirm failure and note where the
   unreachable message comes from
2. Traceroute to find exactly which hop the trail
   breaks at
3. Check `show ip route` on that router for the
   destination network
4. Add the missing route and verify

Four steps. Same process works for almost any
static routing connectivity failure.

---

## Commands Used
```
ping [ip]               → confirm failure, note source of unreachable
tracert/traceroute [ip] → find the breaking hop
show ip route           → check routing table for missing entries
ip route [net] [mask] [next-hop]  → add the missing static route
```
