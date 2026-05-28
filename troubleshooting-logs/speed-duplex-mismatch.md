## What I Was Experimenting With

After manually configuring speed and duplex on the SW1-R2 
link I got curious — what actually happens if the settings 
don't match? Rather than just reading about it I deliberately 
introduced mismatches and watched what the interface status 
output showed on each side. The results were more interesting 
than I expected.

---

## Topology at the Time

<img width="683" height="217" alt="Screenshot 2026-05-28 105854" src="https://github.com/user-attachments/assets/fc9fc422-5bcb-47f2-93d6-88191c0bd104" />

Both the SW1-R2 link had been manually configured at 
100Mbps full duplex on both sides and was working fine 
before I started experimenting.

---

## Experiment 1 — Duplex Mismatch

Set SW1's F0/2 to half duplex while leaving R2 at full duplex:

SW1(config-if)#duplex half

The interface went down immediately:
%LINK-5-CHANGED: Interface FastEthernet0/2, changed state to down
%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/2,
changed state to down

Checked the status:
```
SW1#show ip interface brief
FastEthernet0/2    unassigned    YES manual    down    down
```
Down/down on both status and protocol. The link completely 
refused to come up with mismatched duplex settings. This 
makes sense — half and full duplex have fundamentally 
different ways of handling traffic on the wire. Half duplex 
only sends in one direction at a time, full duplex sends 
in both simultaneously. They can't negotiate a middle ground 
so the link just drops.

Fixed by setting both sides back to full:
```
SW1(config-if)#duplex full
```
Link came straight back up.

---

## Experiment 2 — Speed Mismatch

This one produced something more interesting. Set SW1's 
F0/2 to 10Mbps while R2 stayed at 100Mbps:
```
SW1(config-if)#speed 10
```
Interface dropped again on SW1:
```
%LINK-5-CHANGED: Interface FastEthernet0/2, changed state to down
```
Checked SW1:
```
SW1#show ip interface brief
FastEthernet0/2    unassigned    YES unset    down    down
```
Down/down — expected. But then checked R2:
```
R2#show ip interface brief
FastEthernet0/0    10.10.10.2    YES manual    up    down
```
**Up/down** — R2 thinks the physical layer is fine (Status: 
up) but the line protocol is down. This is the interesting 
part.

From R2's perspective the cable is connected and it can 
detect a signal — so Status shows up. But it can't actually 
communicate with SW1 because they're running at different 
speeds — so Protocol shows down.

From SW1's perspective it can't even detect a valid signal 
at 10Mbps from a device running at 100Mbps — so both Status 
and Protocol show down.

Same broken link. Two completely different views of it 
depending on which end you're looking from.

---

## What The Status Columns Actually Mean

This experiment made the two status columns click in a way 
that reading about them didn't:

| SW1 Status | SW1 Protocol | R2 Status | R2 Protocol | Cause |
|------------|--------------|-----------|-------------|-------|
| up | up | up | up | Everything fine |
| admin down | down | — | — | Manually shut down |
| down | down | up | down | Speed mismatch |
| down | down | down | down | Duplex mismatch or cable issue |

The up/down combination on one side is the tell for a speed 
mismatch — one device can detect a physical signal but can't 
communicate because the speeds don't align.

---

## The Fix
```
SW1(config-if)#speed 100
```
Both sides back to 100Mbps — link came up immediately.

The rule going forward: if manually setting speed or duplex, 
always configure both ends. If one side is manual and the 
other is auto, auto negotiation can sometimes figure it out 
but it's unreliable. Match both sides explicitly or leave 
both on auto.

---

## Commands Used
```
speed [10/100/1000]          → set interface speed manually
duplex [half/full/auto]      → set duplex mode manually
show ip interface brief      → check status and protocol columns
show interface [int]         → detailed view including current speed/duplex
```
