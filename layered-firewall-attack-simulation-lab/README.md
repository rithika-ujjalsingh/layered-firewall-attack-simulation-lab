# Layered Network Defense & Attack Simulation Lab

**A segmented, firewall-protected virtual lab built to practice defense-in-depth architecture, IDS deployment, and controlled exploitation testing against an intentionally vulnerable target — with full detection and blocking analysis.**

---

## Project Overview

This lab simulates a small enterprise network segment: an external-facing perimeter firewall (pfSense), an internal isolated network, a monitoring layer (Suricata IDS), and both attacker and target machines. The goal was twofold — build the defensive infrastructure correctly, and then test it under real attack conditions to see what it actually catches and blocks.

**Environment:** VMware Workstation, fully isolated host-only virtual network (no bridge to the internet or home network)

**Machines:**
| Role | OS | Purpose |
|---|---|---|
| Firewall/Gateway | pfSense 2.8.1-RELEASE | Perimeter firewall, NAT, IDS host |
| Attacker | Kali Linux 2026.1 | Reconnaissance and exploitation |
| Target | Metasploitable2 | Intentionally vulnerable Linux host |

**Skills demonstrated:** network segmentation, firewall rule design, IDS/IPS deployment (Suricata), vulnerability scanning (Nmap), exploitation frameworks (Metasploit), log analysis, and — critically — accurately documenting what worked, what didn't, and why.

---

## Architecture

```
                    ┌─────────────────────────┐
                    │   Host Machine (VMware) │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │   pfSense Firewall        │
                    │   WAN: NAT (192.168.40.x) │
                    │   LAN: 192.168.124.2/24   │
                    │   + Suricata IDS on LAN   │
                    └────────────┬──────────────┘
                                 │
                    Isolated Host-Only Network
                         (192.168.124.0/24)
                                 │
              ┌──────────────────┴──────────────────┐
              │                                       │
   ┌──────────▼──────────┐              ┌────────────▼───────────┐
   │  Kali Linux         │              │  Metasploitable2       │ 
   │  192.168.124.100    │─── attacks──▶│  192.168.124.101      │
   │  (Attacker)         │              │  (Target)              │
   └─────────────────────┘              └────────────────────────┘
```

---

## Part 1 — Network Segmentation & Firewall Build

### What was built
- A dedicated host-only virtual network (VMnet2, `192.168.124.0/24`) with no bridge to the host's real network — the lab cannot leak traffic outside itself
- pfSense deployed as a two-interface VM: WAN (NAT, for updates/package installs) and LAN (the isolated segment)
- LAN configured with a static IP and its own DHCP scope so every lab VM gets a predictable address
- A custom outbound firewall rule (HTTPS, port 443) added and applied through the GUI, on top of the default LAN rules
- Default admin credentials rotated immediately after first login (documented in the hardening section below — this is a real finding auditors check for)

### Safety hardening applied
- VMware Guest Isolation (drag-and-drop, clipboard sharing) disabled on every lab VM
- Clean-state snapshots taken before any exploitation activity, so the lab can be reset instantly
- No lab VM ever set to Bridged networking — Host-only only, confirmed on each VM individually
- Host machine's own firewall verified active as a secondary safety net

**Screenshots:** `01-network-setup/` through `07-firewall-rule/`

---

## Part 2 — IDS Deployment (Suricata)

Suricata was installed as a pfSense package and bound to the LAN interface in IDS mode (detect-only, non-blocking), so it could observe all traffic between the attacker and target without interfering with the exploitation tests.

- Hardware offloading (checksum, TCP segmentation, large receive) disabled per Suricata's requirements — a step frequently missed and worth calling out explicitly in a write-up
- Interface confirmed "running" (green status) before any test traffic was generated

**Screenshots:** `09-suricata-ids/`

---

## Part 3 — Attack Simulation & Detection Analysis

### Reconnaissance
An Nmap service-version scan (`nmap -sV`) was run from Kali against the target, identifying 20+ open services including vsftpd 2.3.4, OpenSSH, Samba, MySQL, PostgreSQL, UnrealIRCd, and distccd — each a known, deliberately vulnerable service on Metasploitable2.

**Result:** Suricata logged the scan as an application-layer protocol anomaly against the target's PostgreSQL port within seconds — confirmed via the Alerts log, source/destination/timestamp matching the scan exactly.

### Exploitation attempts
Four Metasploit modules were run against confirmed-vulnerable services:

| Exploit | Service | Result |
|---|---|---|
| `unix/ftp/vsftpd_234_backdoor` | vsftpd 2.3.4 | Target confirmed vulnerable; backdoor spawned; manual netcat connection to port 6200 succeeded, confirming the backdoor was live |
| `multi/samba/usermap_script` | Samba | Exploit completed; no reverse session established |
| `unix/irc/unreal_ircd_3281_backdoor` | UnrealIRCd | Target confirmed vulnerable during registration handshake; no reverse session established |
| `unix/misc/distcc_exec` | distccd | Exploit completed; no reverse session established |

### Key finding
Every exploit **confirmed the target was vulnerable** at the service-detection stage. However, reverse-shell **callback connections consistently failed to establish a session** back to the attacker, despite the initial backdoor/exploit trigger succeeding on the target side.

This is the most valuable finding in the lab: it is consistent with the pfSense LAN ruleset **not having an explicit "allow all outbound" rule beyond the default LAN-to-any and the added HTTPS rule** — meaning non-standard outbound ports (like Metasploit's default `4444` handler) were not cleanly traversing the firewall/NAT path in this topology. Rather than treating this as a failure, it was documented as evidence that **firewall policy affects attacker tooling in measurable, reproducible ways** — which is exactly the kind of before/after signal a segmentation project is supposed to produce.

**Screenshots:** `10-attack-simulation/`, `11-detection-review/`

---

## Results Summary

| Test | Outcome |
|---|---|
| Network segmentation | Isolated network confirmed — no bridge to host/internet |
| Firewall rule enforcement | Custom rule created, applied, verified in ruleset |
| IDS visibility | Suricata detected reconnaissance scan in real time |
| Exploit delivery | 4/4 exploits confirmed target vulnerability |
| Reverse shell callback | 0/4 established a session — consistent with firewall/NAT behavior on the LAN segment |
| Manual backdoor verification | vsftpd backdoor confirmed live via direct netcat connection to spawned port |

---

## What I'd Do Differently / Next Steps

- Add an explicit "allow all outbound, log everything" rule temporarily to isolate whether the callback failures were firewall-side or NAT/topology-side, then re-test with the restrictive ruleset to get a clean before/after comparison
- Enable the Suricata ET-EXPLOIT and ET-SCAN rule categories (only the default ruleset was active here) and re-run the same attack sequence to measure the increase in detection fidelity
- Add pfSense logging on the specific rule that's dropping the callback traffic, to get a definitive root cause rather than an inferred one
- Map each attack stage to MITRE ATT&CK technique IDs (Reconnaissance = T1595, Exploitation for Client Execution = T1203) for a more formal report structure

---

## Repository Structure

```
├── README.md                          ← this file
├── writeup-detection-analysis.md      ← detailed technical analysis
├── screenshots/
│   ├── 01-network-setup/
│   ├── 02-pfsense-vm/
│   ├── 03-pfsense-install/
│   ├── 04-safety-hardening/
│   ├── 05-kali-connect/
│   ├── 06-pfsense-gui/
│   ├── 07-firewall-rule/
│   ├── 08-target-vm/
│   ├── 09-suricata-ids/
│   ├── 10-attack-simulation/
│   └── 11-detection-review/
```

---

## Disclaimer

All testing was performed exclusively against machines I own and control, inside a fully isolated virtual network with no route to the internet or any third-party system. Metasploitable2 is an intentionally vulnerable OS distributed specifically for this purpose. This project is for educational and portfolio purposes only.
