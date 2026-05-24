# The IOS Operating System

## What This Lab Is About
Getting comfortable with the Cisco IOS command line interface for the first time. This was purely a guided walkthrough — no prior IOS experience needed.

---

## Topology
Single router (Router0) in Packet Tracer. No complex topology here — the focus was entirely on the CLI itself.

---

## Field Notes

### The Three Main Modes

| Mode | Prompt | What you can do here |
|------|--------|----------------------|
| User Exec | `Router>` | Very limited, basic show commands only |
| Privileged Exec | `Router#` | Full show commands, reboot, save config |
| Global Config | `Router(config)#` | Actually change the device configuration |

The most common beginner mistake (which I also hit) — running a command at the wrong mode and getting:
% Invalid input detected at '^' marker.
Before assuming you made a typo, always check which mode you're in.

### Getting Around the CLI

**Abbreviations** — you don't have to type full commands:
Router>en          → enable
Router#conf t      → configure terminal
Router#disa        → disable

Only works when the letters you type match exactly one command. 
Typing `di` gives an ambiguous error because `dir`, `disable` and `disconnect` all start with `di`.

**Tab completion** — type partial command + Tab key and IOS completes it.

**The ? key** — your best friend:
Router#sh ?        → shows all show commands available
Router#sh aaa ?    → shows options specifically for show aaa
Router#di?         → shows all commands starting with 'di'

Note the space before `?` matters:
- `sh?` → tells you what commands start with sh
- `sh ?` → tells you what comes after the show command

**Arrow keys:**
- Up/Down → cycle through command history
- Ctrl+A → jump to beginning of line
- Ctrl+E → jump to end of line
- Ctrl+C or `end` → drop back to Privileged Exec from anywhere

### Running Show Commands from Config Mode
Normally `show` commands only work in Privileged Exec mode. But if you're deep inside config mode and don't want to exit, just add `do` at the start:

R1(config-if)#do show ip interface brief. Works from any level. I use this constantly now.

### Pipe Commands (Filtering Output)
When `show run` dumps a wall of text, these help a lot:
R1#show run | begin hostname      → output starts from the word 'hostname'
R1#show run | include interface   → shows only lines containing 'interface'
R1#show run | exclude interface   → shows everything except 'interface' lines
Important gotcha — pipe commands ARE case sensitive even though 
everything else in IOS is not:
R1#sh run | begin Hostname        → returns nothing (capital H)
R1#sh run | begin hostname        → works correctly

### Paging Through Long Output
When you see `--More--` at the bottom:
- **Enter** → one line at a time
- **Space** → one full page at a time
- **Q** → quit and go back to prompt

---

## Configuration Management — The Important Bit

| Config | Where it lives | What happens to it |
|--------|---------------|-------------------|
| Running config | RAM | Lost on reboot |
| Startup config | NVRAM | Survives reboot |

So any change you make takes effect **immediately** but is **not saved** until you explicitly copy it:

R1#copy run start
Verified this by changing hostname to RouterX, checking startup config 
still showed the old hostname R1, then saving and confirming it updated.

**Backing up config:**
RouterX#copy run flash:      → saves to router's own flash (not ideal)
RouterX#copy run tftp        → saves to external TFTP server (proper way)

---

## Arsenal
```
enable                          → enter Privileged Exec mode
disable                         → drop back to User Exec
configure terminal              → enter Global Config mode
end                             → drop to Privileged Exec from anywhere
exit                            → drop back one level
reload                          → reboot the device
hostname [name]                 → set device hostname
show running-config             → view current active config
show startup-config             → view saved config
copy run start                  → save running config to startup
show ip interface brief         → quick view of all interfaces and status
interface gigabitEthernet 0/0   → enter interface config mode
```
---

## Traps I Fell Into

- The `invalid input` error doesn't always mean a typo — wrong mode is the more likely cause and I kept forgetting this initially.
- Pipe commands being case sensitive while everything else isn't felt inconsistent and caught me off guard.
- The running vs startup config distinction sounds simple but it really clicks only when you actually change something, check startup, and see the old
  value still sitting there.

---

## Onwards To
[Lab 2 - Cisco Device Functions](../cisco-device-functions/README.md)
