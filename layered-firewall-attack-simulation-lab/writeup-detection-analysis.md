# Technical Deep-Dive: Exploitation Behavior & Detection Analysis

This document expands on the findings in `PROJECT-REPORT.md`, walking through the lab build, exploitation attempts, and detection validation in the order they occurred — with each stage mapped to its supporting screenshot.

## 1. Network Foundation

**Screenshot:** `01-vmnet-config.png`

VMnet2 (192.168.124.0/24) was created as a dedicated, isolated segment for this lab, separate from an existing Active Directory lab running on VMnet1. Keeping the two projects on different virtual networks meant traffic and addressing from one could never collide with or affect the other.

## 2. Firewall Interface Design

**Screenshots:** `02-dual-adapters.png`, `03-interfaces-final.png`

The pfSense VM was built with two network interfaces by design: one on NAT (acting as the WAN, giving pfSense internet access for package installation) and one on the isolated VMnet2 (acting as the LAN, facing the attacker and target machines). This two-interface design is what makes it a firewall rather than just another host on the network — all LAN-to-WAN traffic is forced through pfSense's ruleset.

During installation, the two interfaces were manually assigned — `em0` to WAN, `em1` to LAN — with the LAN interface given a static IP (`192.168.124.2/24`) matching the isolated network's subnet, and DHCP enabled on that interface so every future lab VM would receive a predictable address automatically.

## 3. Lab Isolation & Safety Hardening

**Screenshot:** `04-guest-isolation.png`

Before any exploitation testing began, every lab VM had drag-and-drop and clipboard sharing disabled between host and guest, clean-state snapshots were taken so the environment could be reset instantly, and every VM's network adapter was confirmed to be on Host-only/Custom VMnet2 — never Bridged. This is standard lab hygiene: it prevents accidental data leakage between the isolated lab and the host machine, and guarantees a fast, reliable reset point if a VM is left in a bad state after an exploitation attempt.

## 4. Attacker & Target Provisioning

**Screenshots:** `05-kali-ip.png`, `08-target-ip.png`

Kali Linux (attacker) was confirmed on `192.168.124.100` and Metasploitable2 (target) on `192.168.124.101`, both addressed via the LAN-side DHCP scope configured on pfSense — confirming the addressing plan worked exactly as designed before any attack traffic was generated.

## 5. Firewall Rule & Dashboard Verification

**Screenshots:** `06-dashboard.png`, `07-rule-applied.png`

The pfSense web GUI dashboard was confirmed reachable and operational, and a custom firewall rule was added and applied on the LAN interface. Verifying the rule was live (rather than just saved) before proceeding ensured that all subsequent attack traffic actually passed through an enforced ruleset, not a default-allow configuration.

## 6. Reconnaissance

**Screenshot:** `11-nmap-results.png`

A full Nmap service/version scan from Kali against `192.168.124.101` identified 20+ open ports, consistent with Metasploitable2's intentionally vulnerable service footprint. This scan is also the reference event used later for detection validation (see Section 8).

## 7. Exploitation Attempts & the Session-Establishment Gap

**Screenshot:** `12-vsftpd-exploit.png`

Four Metasploit modules were run against confirmed-vulnerable services on the target:

- `exploit/unix/ftp/vsftpd_234_backdoor`
- `exploit/multi/samba/usermap_script`
- `exploit/unix/irc/unreal_ircd_3281_backdoor`
- `exploit/unix/misc/distcc_exec`

All four modules confirmed the target was vulnerable — each exploit successfully triggered its target-side condition. None, however, returned an interactive Meterpreter/shell session back through Metasploit. In a NAT'd lab topology, this is a known and explainable outcome rather than evidence the vulnerabilities were false: the backdoor process on the target can spawn correctly, but the automated session handler's callback path is sensitive to how NAT and the firewall's stateful rules handle the resulting connection. Treating "exploit reports success, no session" as inconclusive — rather than as either "vulnerable" or "not vulnerable" — was the right call here, which is why the next step was independent, manual verification.

### Manual Verification

**Screenshot:** `10-backdoor-verify.png`

To resolve the ambiguity, the vsftpd backdoor was checked directly with raw `netcat`, independent of Metasploit's own session handling:

```
nc -nv 192.168.124.101 6200
```

The connection returned an **open** state on port 6200 within seconds of the exploit spawning it — this screenshot was captured immediately after the Metasploit `run` command completed, since the backdoor listener closes quickly if not caught right away. This confirmed the vulnerability was genuinely present and reachable on the target, isolating the earlier session-handling failure to the network path rather than the exploit itself.

## 8. Detection Validation

**Screenshot:** `09-suricata-running.png`, `13-suricata-alert.png`

Suricata was confirmed running on the pfSense LAN interface, giving it visibility into all attacker-to-target traffic on the segment. Cross-referencing the Suricata alert log against the exact timestamp of the Nmap scan (Section 6) confirmed real-time detection — the IDS flagged the reconnaissance activity within seconds, using only the default (non-extended) ruleset.

This is the core deliverable of the lab's detection phase: not just that an attack was attempted, but documented, timestamped proof of what the defensive layer actually saw and logged while it happened.

## Summary of Root-Cause Reasoning

| Observation | Interpretation |
|---|---|
| Exploit modules report target-side success | Vulnerability is real |
| No interactive session returned via Metasploit | Network-layer (NAT/firewall) interaction with the callback, not a false positive |
| Manual `netcat` confirms backdoor port open | Independently verifies the vulnerability, isolated from framework session handling |
| Suricata logs the scan within seconds of it occurring | Detection layer is functioning correctly at baseline, default configuration |

This progression — automated attempt → ambiguous result → manual independent verification → correlated detection evidence — reflects the kind of investigative process expected in real security testing, where a single tool's output is treated as one data point rather than the final answer.
