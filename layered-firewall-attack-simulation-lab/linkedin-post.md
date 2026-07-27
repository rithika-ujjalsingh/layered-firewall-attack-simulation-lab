# LinkedIn Post — Full Project

🔒 New project: Layered Network Defense & Attack Simulation Lab

Built a segmented home lab to test defense-in-depth architecture end-to-end — not just theory, but actually attacking it and measuring what the defenses caught.

What I built:
→ Isolated virtual network with pfSense as a dedicated perimeter firewall
→ Suricata IDS deployed and monitoring the internal segment in real time
→ An intentionally vulnerable target host and a Kali attacker on the same segment

What I tested:
→ Ran reconnaissance (Nmap) — Suricata flagged it within seconds, alert log matched the scan exactly
→ Attempted exploitation against 4 confirmed-vulnerable services (vsftpd, Samba, UnrealIRCd, distcc)
→ Every exploit confirmed the target was vulnerable — but reverse-shell callbacks consistently failed to establish a session back through the firewall

The most useful finding wasn't "the exploit worked" — it was that the firewall's presence measurably interfered with post-exploitation callback traffic, even without a rule explicitly written to block it. That's the actual signal a segmentation project is supposed to produce, and I documented the full root-cause analysis rather than treating it as a failed test.

Full write-up, architecture diagram, and screenshots on GitHub: [add your repo link]

#cybersecurity #networksecurity #pfsense #idsips #homelab #infosec #purpleteam
