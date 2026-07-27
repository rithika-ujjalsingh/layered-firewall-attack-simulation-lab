# Screenshot Checklist — What Goes in Each Folder

Use this as the final check before uploading to GitHub. Each folder should contain only the screenshots that prove that specific step — not every screenshot taken during troubleshooting. Rename files to match the suggested names below; consistent naming is itself a professional signal.

## 01-network-setup/
- [ ] `step01-vmnet2-hostonly-config.png` — Virtual Network Editor showing VMnet2, Host-only, DHCP unchecked

## 02-pfsense-vm/
- [ ] `step02-vm-created.png` — pfSense-Firewall VM created in VMware library
- [ ] `step03-dual-adapters.png` — Settings showing both network adapters (NAT + VMnet2)

## 03-pfsense-install/
- [ ] `step04-interfaces-final.png` — Console showing final WAN/LAN assignment with IPs
  *(`WAN -> em0 -> v4/DHCP4: ...` and `LAN -> em1 -> v4: 192.168.124.2/24`)*

## 04-safety-hardening/
- [ ] `step05-guest-isolation.png` — Guest Isolation settings (drag/drop + clipboard unchecked)
- [ ] `step06-host-firewall-on.png` — Windows Security firewall status, all profiles ON

## 05-kali-connect/
- [ ] `step07-kali-ip-assigned.png` — Kali terminal `ip a` output showing `192.168.124.100`

## 06-pfsense-gui/
- [ ] `step08-pfsense-dashboard.png` — pfSense web dashboard after login

## 07-firewall-rule/
- [ ] `step09-rule-created.png` — LAN rules list showing the custom HTTPS rule, before Apply
- [ ] `step10-rule-applied.png` — Green "changes applied successfully" banner

## 08-target-vm/
- [ ] `step11-target-ip.png` — Metasploitable terminal `ifconfig` showing `192.168.124.101`

## 09-suricata-ids/
- [ ] `step12-suricata-running.png` — Suricata Interfaces tab, LAN row, green running status

## 10-attack-simulation/
- [ ] `step13-nmap-results.png` — Full Nmap scan output (20+ ports)
- [ ] `step14-vsftpd-exploit-attempt.png` — msfconsole vsftpd exploit run showing "Backdoor has been spawned"
- [ ] `step15-manual-backdoor-verify.png` — netcat connection to port 6200 showing "open"
- [ ] `step16-unrealircd-vulnerable.png` — msfconsole output confirming "target appears to be vulnerable"

## 11-detection-review/
- [ ] `step17-suricata-alert-scan.png` — Suricata Alerts tab showing the logged Nmap scan alert
  *(source 192.168.124.100 → dest 192.168.124.101, timestamp matching the scan)*

---

## What NOT to include
Skip screenshots of: login error messages, repeated failed attempts at the same step, typo corrections, or anything mid-troubleshooting. GitHub viewers want the clean proof trail, not the debugging process — that narrative belongs in the write-up text, not the image folder.

## Before uploading
1. Open each folder and confirm every image actually shows what its filename claims
2. Blur or crop out anything showing your real IP, MAC address if you're concerned about that, or any personal file paths visible in title bars
3. Compress large PNGs if the repo is getting heavy (GitHub soft-caps around 1GB per repo)
