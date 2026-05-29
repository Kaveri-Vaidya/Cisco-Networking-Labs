## What this lab is about
This lab is about the operational side of managing Cisco devices 
— the stuff that matters when things go wrong in the real world. 
Factory resets, password recovery, configuration backups, IOS 
image backup and upgrade. Knowing CLI commands is one thing but 
knowing how to recover a locked-out router or restore a device 
from scratch is a completely different level of practical skill. 

---

## Topology

<img width="725" height="313" alt="Screenshot 2026-05-28 224442" src="https://github.com/user-attachments/assets/4397c99c-873a-4ae7-b912-8feeb9d2a044" />

---

## Preparation — Backing Up Config to TFTP First

Before doing anything destructive I backed up R1's running 
configuration to the TFTP server. This is the step that 
makes everything else recoverable:
```
R1#copy running-config tftp
Address or name of remote host []? 10.10.10.10
Destination filename [R1-confg]? R1-running-config
```
Verified it landed on the tftp server:

<img width="576" height="223" alt="Screenshot 2026-05-29 172932" src="https://github.com/user-attachments/assets/635bf4bc-5528-4319-a8ad-6ffb71b3657e" />

This is also exactly what you'd do in a real network before 
any maintenance window — backup first, then proceed. 
Nobody should be touching production devices without a 
config backup sitting somewhere safe.

---

## Field Notes

### Factory Reset

A factory reset wipes everything — startup config, running 
config, all of it. The device comes back up as if it just 
came out of the box.
```
R1#write erase
Erasing the nvram filesystem will remove all configuration files! Continue? [confirm]
R1#reload
```
After reboot the router launched the Setup Wizard — which 
is what happens when it finds no startup config. Exited 
out of it, confirmed both configs were empty:
```
R1#show startup-config
startup-config is not present
R1#show running-config
```
Only default entries — hostname Router, nothing else.

**When you'd actually do this in real life:**
Decommissioning a device before returning it to a vendor, 
sending hardware to a different site, or wiping a router 
that's being repurposed for a completely different role. 
You never want the old configuration sitting on hardware 
that's leaving your hands.

Restored R1's config by pulling it back from the TFTP 
server:
```
Router#copy tftp running-config
Address or name of remote host []? 10.10.10.10
Source filename []? R1-backup
```
Saved it:
```
R1#copy running-config startup-config
```
---

### Password Recovery

This is the scenario where someone has set an enable 
secret on a router and either left the company, forgotten 
it, or simply never documented it. The router is locked 
and you need back in without wiping it.

The process works by telling the router to skip loading 
the startup config on boot — so it comes up with no 
password. Then you copy the startup config in manually, 
remove the password, and restore normal boot behaviour.

**Step 1 — Set a password to simulate the lockout:**
```
R1(config)#enable secret CiscoDev1
R1#copy run start
```
**Step 2 — Force boot into rommon:**
```
R1(config)#config-register 0x2100
R1#reload
```
rommon is the router's low-level bootstrap environment — 
it runs before IOS loads. From here:
```
rommon 1 > confreg 0x2142
rommon 2 > reset
```
`0x2142` tells the router to ignore the startup config 
on next boot. The router comes up into the Setup Wizard 
again — exit out of it.

**Step 3 — The critical step most people miss:**

At this point the running config is empty and the startup 
config still has the password in it. Do NOT save anything 
yet. Copy startup to running first:
```
Router#copy startup-config running-config
```
Now the full config is loaded including the password — 
but you're already in privileged exec so the password 
doesn't stop you. Remove it:
```
R1(config)#no enable secret
```
**Step 4 — Restore normal boot behaviour:**
```
R1(config)#config-register 0x2102
R1#copy run start
R1#reload
```
Router comes back up normally, no password, full config 
intact.

**When you'd actually do this in real life:**
Any time access to a device is lost — staff turnover, 
undocumented passwords, inherited infrastructure with 
no documentation. This is a standard procedure that 
network engineers need to know cold. Skipping the 
`copy startup-config running-config` step before 
removing the password is the mistake that turns a 
password recovery into an accidental factory reset.

---

### Configuration Backup

Two places to back up to — local flash and external TFTP:

**To flash (local backup):**
```
R1#copy running-config flash:
Destination filename [running-config]? R1-config-backup
```
Verified:
```
R1#show flash
```
**To TFTP server (proper backup):**
```
R1#copy startup-config tftp
Address or name of remote host []? 10.10.10.10
Destination filename [R1-confg]? R1-startup-backup
```
Flash backup is better than nothing but it's on the same 
device — if the hardware fails you lose the backup too. 
TFTP backup to a separate server is the real safeguard.

**When you'd actually do this in real life:**
Before any configuration change in production. Before 
a maintenance window. As part of a scheduled automated 
backup job. Config backups are what allow you to roll 
back a bad change without panicking.

---

### IOS System Image Backup and Recovery

The IOS image is the operating system itself. Backing it 
up means you can recover a router that has lost its image 
due to corruption or a failed upgrade.

**Backup the image to TFTP:**
```
R1#copy flash: tftp
Source filename []? c2900-universalk9-mz.SPA.151-4.M4.bin
Address or name of remote host []? 10.10.10.10
Destination filename [c2900-universalk9-mz.SPA.151-4.M4.bin]?
```
**Simulated recovery — deleted the image then restored:**
```
R1#delete flash:c2900-universalk9-mz.SPA.151-4.M4.bin
R1#reload
```
Router boots into rommon — no IOS to load. From rommon, 
restored the image over TFTP:
```
rommon 1 > IP_ADDRESS=10.10.10.1
rommon 2 > IP_SUBNET_MASK=255.255.255.0
rommon 3 > DEFAULT_GATEWAY=10.10.10.2
rommon 4 > TFTP_SERVER=10.10.10.10
rommon 5 > TFTP_FILE=c2900-universalk9-mz.SPA.151-4.M4.bin
rommon 6 > tftpdnld
```
Router pulled the image from TFTP, wrote it to flash, 
and booted normally.

**When you'd actually do this in real life:**
IOS corruption can happen — power failures mid-write, 
failed upgrades, hardware faults. Having the image backed 
up externally is the difference between a 10 minute 
recovery and ordering replacement hardware. Large 
organisations maintain a dedicated image repository 
with every IOS version they run across their fleet.

---

### IOS Image Upgrade on SW1

Verified what SW1 was running first:
```
SW1#show version
Cisco IOS Software, C2960 Software (C2960-LANBASE-M),
Version 12.2(25)FX
```
Copied the new image from TFTP to flash:
```
SW1#copy tftp flash
Address or name of remote host []? 10.10.10.10
Source filename []? c2960-lanbasek9-mz.150-2.SE4.bin
Destination filename [c2960-lanbasek9-mz.150-2.SE4.bin]?
```
Told the switch to boot from the new image:
```
SW1(config)#boot system flash:c2960-lanbasek9-mz.150-2.SE4.bin
SW1#copy run start
SW1#reload
```
Verified after reboot:
```
SW1#show version
Cisco IOS Software, C2960 Software (C2960-LANBASEK9-M),
Version 15.0(2)SE4
```
**When you'd actually do this in real life:**
IOS upgrades happen for security patches, bug fixes, or 
new feature requirements. The process is always the same 
— copy the new image to flash first, verify it's there, 
then tell the device to boot from it. Never delete the 
old image until the new one is confirmed working. 
Organisations with large switch fleets use tools like 
Cisco DNA Center to automate this across hundreds of 
devices at once — but the underlying process is identical 
to what's done here manually.

---

## Arsenal
```
write erase                          → wipe the startup config (factory reset)
reload                               → reboot the device
copy running-config tftp             → backup config to TFTP server
copy tftp running-config             → restore config from TFTP server
copy running-config flash:           → backup config to local flash
copy startup-config running-config   → load saved config into running config
copy flash: tftp                     → backup IOS image to TFTP
copy tftp flash:                     → copy IOS image from TFTP to flash
show flash                           → view contents of flash memory
show version                         → check current IOS version
config-register 0x2100               → force boot into rommon on next reload
config-register 0x2142               → ignore startup config on next boot
config-register 0x2102               → normal boot behaviour
boot system flash:[filename]         → specify which IOS image to boot from
enable secret [password]             → set encrypted enable password
no enable secret                     → remove enable password
```
---

## Traps I Fell Into

- During password recovery, copying startup to running 
  before removing the password is the step that's easy 
  to forget — and skipping it means saving an empty config 
  over your startup config. That's an accidental factory 
  reset. The order matters: copy startup to running first, 
  then remove the password, then save.

- config-register values look like arbitrary hex numbers 
  but each bit means something specific. The two that 
  matter most are 0x2142 (ignore startup config) and 
  0x2102 (normal boot). Forgetting to set it back to 
  0x2102 after password recovery means the router skips 
  the startup config on every subsequent reboot — the 
  device appears to lose its config every time it restarts.

---

## Onwards To
[Routing Fundamentals](../6.%20routing-fundamentals/README.md)
