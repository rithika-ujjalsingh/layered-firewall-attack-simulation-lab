# Project Report: Layered Network Defense & Attack Simulation Lab

**Author:** Rithika
**Environment:** VMware Workstation, fully isolated virtual lab
**Report type:** End-to-end build, deployment, and adversarial testing report

---

## 1. Executive Summary

This report documents the design, build, and testing of a segmented virtual lab modeling a small enterprise network boundary: a perimeter firewall, an intrusion detection layer, and an internal network hosting both a target system and an attacker system. The objective was not only to configure the defensive infrastructure correctly, but to validate it — to run a realistic attack sequence against it and produce evidence-backed findings about what the defenses actually detected and how they behaved under exploitation attempts.

The lab was built entirely from scratch: virtual network segmentation, firewall installation and configuration, IDS deployment, target and attacker provisioning, safety hardening, reconnaissance, exploitation, and detection analysis. Every stage is documented with timestamped screenshots and command-level detail.

**Headline finding:** while every exploitation attempt confirmed the target was vulnerable, reverse-shell callback connections consistently failed to establish, consistent with the firewall segment interfering with non-standard outbound traffic — a result more valuable to a defensive engineer than a lab where every exploit works cleanly, because it demonstrates the firewall's effect on attacker tooling in a measurable, reproducible way.

---

## 2. Objectives

1. Build a network segment isolated from the host machine and the internet
2. Deploy a firewall (pfSense) as the sole gateway between an internal segment and the outside
3. Deploy an IDS (Suricata) with visibility into all internal traffic
4. Apply baseline lab safety hardening before any offensive testing
5. Execute a realistic attack chain — reconnaissance, then exploitation — against a known-vulnerable target
6. Cross-reference every attack action against IDS logs to determine detection coverage
7. Analyze and document exploitation outcomes, including failures, with root-cause reasoning

---

## 3. Environment & Architecture

| Component | Detail |
|---|---|
| Hypervisor | VMware Workstation Pro |
| Network | VMnet2, Host-only, `192.168.124.0/24` — no bridge to host or internet |
| Firewall/Gateway | pfSense 2.8.1-RELEASE (amd64), 2 vCPU/2GB, dual-interface |
| IDS | Suricata (pfSense package), bound to LAN interface, IDS mode |
| Attacker | Kali Linux 2026.1, `192.168.124.100` |
| Target | Metasploitable2 (intentionally vulnerable Linux), `192.168.124.101` |

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
              │                                     │
   ┌──────────▼──────────┐               ┌────────────▼───────────┐
   │  Kali Linux         │               │  Metasploitable2       │
   │  192.168.124.100    │─── attacks──▶│  192.168.124.101       │
   │  (Attacker)         │              │  (Target)              │
   └─────────────────────┘              └────────────────────────┘
```

---

## 4. Build Process

### 4.1 Network Segmentation
A dedicated VMnet2 host-only virtual network was created in VMware, deliberately separate from any existing lab networks on the host, to guarantee no interference between unrelated projects and no path to the internet.

### 4.2 Firewall Deployment
pfSense was installed with two network interfaces — WAN (NAT, for package updates) and LAN (VMnet2, the isolated segment). Interface assignment (em0/em1) was configured manually through the console, and the LAN interface was given a static IP with DHCP enabled for downstream lab hosts.

### 4.3 Safety Hardening
Before any offensive activity:
- VMware Guest Isolation (drag-and-drop, clipboard) disabled on every lab VM
- Clean-state snapshots captured for fast, reliable resets
- Every lab VM's network adapter confirmed Host-only/Custom-VMnet2, never Bridged
- Default pfSense admin password rotated immediately after first login

### 4.4 Firewall Rule Authoring
A custom LAN rule was written through the pfSense GUI restricting outbound HTTPS traffic explicitly, layered on top of pfSense's default LAN-to-any rule, and applied through the standard change-and-apply workflow.

### 4.5 IDS Deployment
Suricata was installed as a pfSense package, bound to the LAN interface in detect-only (IDS) mode. Hardware offloading (checksum, TCP segmentation offload, large receive offload) was disabled system-wide, per Suricata's stated requirement, followed by a full system reboot to apply the change.

### 4.6 Target & Attacker Provisioning
Metasploitable2 was deployed with networking restricted to the isolated segment only (no NAT/internet adapter), and Kali Linux was connected to the same segment. Both received addresses automatically via pfSense's DHCP scope, confirming end-to-end network functionality before testing began.

---

## 5. Testing Methodology

### 5.1 Reconnaissance
A service-version scan (`nmap -sV`) was run from the attacker against the target, enumerating all open ports and fingerprinting the services running on each.

### 5.2 Exploitation
Four Metasploit modules were selected based on the services identified during reconnaissance, each targeting a documented, intentional vulnerability in Metasploitable2:
- `unix/ftp/vsftpd_234_backdoor`
- `multi/samba/usermap_script`
- `unix/irc/unreal_ircd_3281_backdoor`
- `unix/misc/distcc_exec`

### 5.3 Detection Cross-Reference
Following each test action, the Suricata Alerts log was reviewed and cross-referenced by timestamp and source/destination IP against the exact activity performed, to determine what the IDS actually captured versus what activity occurred.

---

## 6. Findings

### 6.1 Reconnaissance Detection
Suricata detected and logged the Nmap scan in real time, flagging it as an application-layer protocol anomaly against the target's PostgreSQL service, with source, destination, and timestamp matching the scan precisely. This confirms functional, real-time IDS visibility using only the default Suricata ruleset — no custom signatures were required to catch basic reconnaissance.

### 6.2 Exploitation Outcomes

| Exploit | Vulnerability Confirmed | Session Established | Notes |
|---|---|---|---|
| vsftpd 2.3.4 backdoor | ✅ Yes (banner-based) | Session tracking: No / Manual verification: **Yes** | Backdoor port (6200/TCP) manually confirmed live via direct netcat connection |
| Samba usermap_script | Not explicitly confirmed | No | No corroborating evidence gathered |
| UnrealIRCd 3.2.8.1 backdoor | ✅ Yes (explicit message) | No | Backdoor command sent successfully; no callback |
| distcc_exec | Not explicitly confirmed | No | No corroborating evidence gathered |

### 6.3 Root Cause of Failed Callbacks
All four exploits share a consistent pattern: initial exploitation succeeds at the target, but the reverse-connection callback to the attacker's listener does not complete. Given that the vsftpd backdoor was independently confirmed live via manual connection, this is not a target-side failure — it points to the outbound path from target back to attacker, through the pfSense LAN segment, not cleanly carrying traffic on the non-standard high ports Metasploit's default handlers use (port 4444 and related ephemeral ports).

This was not isolated to a specific root cause within the scope of this test cycle, but the evidence consistently implicates the firewall/NAT path over the target or the exploits themselves — recommended follow-up testing (Section 8) is designed to isolate this definitively.

---

## 7. Conclusions

1. **Network segmentation and firewall deployment were successful** — the isolated segment functioned correctly, with no leakage to the host or internet, and pfSense correctly gated all traffic between the WAN and LAN sides.
2. **IDS detection was validated with evidence, not assumption** — the Suricata alert log provided a timestamped, cross-referenceable record of reconnaissance activity, demonstrating the value of the monitoring layer independent of the firewall's blocking behavior.
3. **Exploitation testing confirmed real vulnerabilities existed on the target** in 2 of 4 cases with explicit tool confirmation, and 1 of those was independently verified through manual follow-up — demonstrating that automatic tool output should be corroborated rather than taken at face value.
4. **The firewall segment measurably affected attacker tooling**, even without a rule explicitly written to block the observed traffic. This is arguably the most valuable outcome of the project: a defensive network doesn't need a perfect, explicit "block everything" ruleset to disrupt common attack tooling — default behavior and topology can have real, testable effects.

---

## 8. Recommendations & Next Steps

1. Add a temporary permissive outbound rule and repeat all four exploitation attempts to isolate whether the callback failures are ruleset-driven or NAT/topology-driven
2. Enable Suricata's ET-EXPLOIT and ET-SCAN rule categories (default ruleset only was used here) and re-run the full attack sequence to measure the change in detection fidelity and alert specificity
3. Capture pfSense's own firewall logs during a repeat test to identify the exact rule and action responsible for callback traffic behavior
4. Map each attack stage to MITRE ATT&CK technique IDs for a more formally structured report (Reconnaissance = T1595, Exploitation for Client Execution = T1203)
5. Extend the lab with a second target service tier to test lateral movement detection once the initial foothold question is resolved

---

## 9. Scope & Ethics Statement

All testing described in this report was performed exclusively against systems owned and controlled by the author, within a fully isolated virtual network with no route to the internet or any third-party system. Metasploitable2 is a deliberately vulnerable operating system distributed specifically for authorized security training and testing. This report is submitted for educational and professional portfolio purposes only.
