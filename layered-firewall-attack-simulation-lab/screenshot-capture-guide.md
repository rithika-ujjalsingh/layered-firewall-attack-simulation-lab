# Screenshot Capture Guide — Folder by Folder

Since the lab is already built, most of these screenshots can be captured right now by revisiting each screen (power on the VM, log in, navigate to the page). This guide tells you exactly what to open, what to capture, what to name it, and gives you a one-paragraph write-up to accompany that folder if you want per-section documentation on GitHub.

---

## 📁 01-network-setup

**What to capture:** VMware's Virtual Network Editor showing VMnet2 configured as Host-only.

**How to get there:**
1. Open VMware Workstation
2. Edit → Virtual Network Editor
3. Click VMnet2 in the list so its settings show at the bottom
4. Take the screenshot with the whole window visible — network list at top, Host-only radio button and subnet fields visible at the bottom

**Save as:** `screenshots/01-vmnet-config.png`

**Write-up:**
> A dedicated host-only virtual network (VMnet2) was created to isolate the lab from the host machine's real network and the internet. Host-only mode means VMware routes traffic only between VMs on this virtual switch and the host itself — there is no bridge to a physical NIC, so nothing on this segment can reach or be reached by anything outside the lab.

---

## 📁 02-pfsense-vm

**What to capture:** pfSense VM settings showing both network adapters.

**How to get there:**
1. VMware sidebar → right-click **pfSense-Firewall** → Settings
2. With the Hardware tab open, make sure both "Network Adapter" and "Network Adapter 2" are visible in the device list, along with their type (NAT / Custom VMnet2)

**Save as:** `screenshots/02-dual-adapters.png`

**Write-up:**
> The firewall VM was built with two network interfaces by design: one on NAT (acting as the WAN, giving pfSense internet access for package installation) and one on the isolated VMnet2 (acting as the LAN, facing the attacker and target machines). This two-interface design is what makes it a firewall rather than just another host on the network — all LAN-to-WAN traffic is forced through pfSense's ruleset.

---

## 📁 03-pfsense-install

**What to capture:** The pfSense console screen showing the final WAN/LAN interface assignment with IP addresses.

**How to get there:**
1. Power on the pfSense-Firewall VM (if not already running)
2. Wait for it to fully boot to the console menu
3. Screenshot the section showing:
   ```
   WAN (wan) -> em0 -> v4/DHCP4: 192.168.40.147/24
   LAN (lan) -> em1 -> v4: 192.168.124.2/24
   ```

**Save as:** `screenshots/03-interfaces-final.png`

**Write-up:**
> pfSense was installed and its two interfaces were manually assigned — em0 to WAN, em1 to LAN — with the LAN interface given a static IP (192.168.124.2/24) matching the isolated network's subnet, and DHCP enabled on that interface so every future lab VM would receive a predictable address automatically.

---

## 📁 04-safety-hardening

**What to capture:** VMware's Guest Isolation settings showing drag-and-drop and clipboard sharing disabled.

**How to get there:**
1. Right-click any lab VM (Kali or Metasploitable) → Settings
2. Click the **Options** tab → **Guest Isolation**
3. Screenshot showing both checkboxes unchecked

**Save as:** `screenshots/04-guest-isolation.png`

**Write-up:**
> Before any exploitation testing began, every lab VM had drag-and-drop and clipboard sharing disabled between host and guest, clean-state snapshots were taken so the environment could be reset instantly, and every VM's network adapter was confirmed to be on Host-only/Custom VMnet2 — never Bridged. This is standard lab hygiene: it prevents accidental data leakage between the isolated lab and the host machine, and guarantees a fast, reliable reset point.

---

## 📁 05-kali-connect

**What to capture:** Kali terminal output of `ip a` showing the assigned IP.

**How to get there:**
1. Power on Kali, log in
2. Open a terminal
3. Type `ip a`, press Enter
4. Screenshot the eth0 section showing `inet 192.168.124.100/24`

**Save as:** `screenshots/05-kali-ip.png`

**Write-up:**
> With Kali's network adapter pointed at the same isolated VMnet2 segment as pfSense's LAN interface, Kali received an IP address automatically from pfSense's DHCP server — confirming the firewall's DHCP service and the network path between attacker and gateway were both functioning correctly before any testing began.

---

## 📁 06-pfsense-gui

**What to capture:** The pfSense web dashboard after a successful login.

**How to get there:**
1. From Kali, open Firefox
2. Navigate to `https://192.168.124.2`
3. Accept the self-signed certificate warning
4. Log in with the admin credentials
5. Screenshot the Status/Dashboard page

**Save as:** `screenshots/06-dashboard.png`

**Write-up:**
> With the network path confirmed, the pfSense web administration interface was accessed from inside the isolated LAN segment — the same vantage point an attacker who breached the perimeter would have — to configure firewall rules and enable IDS monitoring for the rest of the lab.

---

## 📁 07-firewall-rule

**What to capture:** The LAN firewall rules list showing the custom rule, and the "changes applied successfully" confirmation.

**How to get there:**
1. In pfSense: Firewall → Rules → LAN tab
2. Screenshot the rules table showing the custom "Allow HTTPS outbound - lab rule" entry
3. If you re-trigger a change and see the green "changes applied successfully" banner, grab that too

**Save as:** `screenshots/07-rule-applied.png`

**Write-up:**
> A custom outbound rule was added on top of pfSense's default LAN ruleset, restricting one class of traffic (HTTPS, port 443) explicitly rather than relying only on the default "allow all" rule. This demonstrates hands-on firewall rule authoring through the GUI — action, protocol, source, destination, port range, and a descriptive label — which later became directly relevant when analyzing why certain outbound exploit callback traffic behaved differently than expected during the attack simulation phase.

---

## 📁 08-target-vm

**What to capture:** Metasploitable2 terminal output of `ifconfig` showing its assigned IP.

**How to get there:**
1. Power on the Metasploitable2-Linux VM
2. Log in (msfadmin / msfadmin)
3. Type `ifconfig`, press Enter
4. Screenshot the eth0 section showing `inet addr:192.168.124.101`

**Save as:** `screenshots/08-target-ip.png`

**Write-up:**
> Metasploitable2 — an intentionally vulnerable Linux distribution built specifically for penetration-testing practice — was deployed on the same isolated segment, network-adapter-only (no internet access), and confirmed to have received an address from pfSense's DHCP scope, completing the three-machine lab topology: firewall, attacker, target.

---

## 📁 09-suricata-ids

**What to capture:** The Suricata Interfaces tab showing the LAN interface with a green "running" status.

**How to get there:**
1. In pfSense: Services → Suricata → Interfaces
2. Screenshot the table row for LAN (em1) showing the green status icon

**Save as:** `screenshots/09-suricata-running.png`

**Write-up:**
> Suricata was installed as a pfSense package and bound to the LAN interface in IDS (detect-only) mode, giving the firewall full visibility into every packet crossing between the attacker and target machines without interfering with the traffic itself. Hardware offloading settings (checksum, TCP segmentation, large receive) were disabled system-wide first, since Suricata explicitly requires this to inspect packets correctly — a step that's easy to miss and worth calling out.

---

## 📁 10-attack-simulation

**What to capture:** Three separate screenshots — the Nmap scan, the vsftpd exploit attempt, and the manual backdoor verification.

**How to get there:**
1. **Nmap:** In Kali, run `nmap -sV 192.168.124.101`, screenshot the full port list output
2. **Exploit attempt:** In msfconsole, run the vsftpd_234_backdoor module, screenshot the output showing "Backdoor has been spawned!"
3. **Manual verification:** In a separate terminal, run `nc -nv 192.168.124.101 6200` right after triggering the exploit, screenshot the "open" connection confirmation

**Save as:**
- `screenshots/10-nmap-results.png`
- `screenshots/11-vsftpd-exploit.png`
- `screenshots/12-backdoor-verify.png`

**Write-up:**
> A standard reconnaissance scan identified 20+ open services on the target, several of which are deliberately vulnerable by design (vsftpd 2.3.4, Samba, UnrealIRCd, distccd). Metasploit modules were run against each of these; every module's automatic vulnerability check confirmed the target was exploitable, and the vsftpd backdoor was independently verified by manually connecting to its spawned listener port — confirming successful exploitation at the target level even where Metasploit's own session tracking didn't register it.

---

## 📁 11-detection-review

**What to capture:** The Suricata Alerts page showing the logged detection of the Nmap scan.

**How to get there:**
1. In pfSense: Services → Suricata → Alerts
2. Set "Instance to View" to LAN
3. Screenshot the alert entry showing source 192.168.124.100 → destination 192.168.124.101, matching the scan's timestamp

**Save as:** `screenshots/13-suricata-alert.png`

**Write-up:**
> Cross-referencing the Suricata alert log against the exact timestamp of the Nmap scan confirmed real-time detection — the IDS flagged the reconnaissance activity within seconds, using only the default (non-extended) ruleset. This is the core deliverable of the lab: not just that an attack was attempted, but documented, timestamped proof of what the defensive layer actually saw and logged while it happened.

---

## Quick Reference Table

| Folder | Screenshot filename(s) | What it proves |
|---|---|---|
| 01-network-setup | `01-vmnet-config.png` | Network isolation exists |
| 02-pfsense-vm | `02-dual-adapters.png` | Firewall has separate WAN/LAN |
| 03-pfsense-install | `03-interfaces-final.png` | Firewall correctly configured |
| 04-safety-hardening | `04-guest-isolation.png` | Lab safety practices followed |
| 05-kali-connect | `05-kali-ip.png` | Attacker correctly placed on segment |
| 06-pfsense-gui | `06-dashboard.png` | Admin access to firewall confirmed |
| 07-firewall-rule | `07-rule-applied.png` | Custom rule authored and applied |
| 08-target-vm | `08-target-ip.png` | Target correctly placed on segment |
| 09-suricata-ids | `09-suricata-running.png` | IDS actively monitoring |
| 10-attack-simulation | `10-nmap-results.png`, `11-vsftpd-exploit.png`, `12-backdoor-verify.png` | Attack sequence executed and verified |
| 11-detection-review | `13-suricata-alert.png` | Detection proven with evidence |
