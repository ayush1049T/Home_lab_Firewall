# 🛡️ Network Security & Intrusion Lab (pfSense + Ubuntu + Kali) [firewall]

A virtualized home-lab environment simulating a perimeter firewall setup. This repository documents the network topology, pfSense firewall rules, and attack/defense scenarios where **pfSense** protects an internal **Ubuntu Server** (`192.168.1.100`) from external attacks launched by a **Kali Linux** machine on the bridged WAN segment.

---

## 🏗️ Network Architecture

The lab isolates the internal Ubuntu server behind the pfSense firewall. Kali Linux sits on the bridged adapter (WAN side), forcing all reconnaissance and attack traffic to pass through pfSense's packet filter and IDS/IPS engine.

```text
+-----------------------+          +-----------------------+          +-----------------------+
|      Kali Linux       |          |        pfSense        |          |     Ubuntu Server     |
|    (Attacker/WAN)     |          |      (Firewall)       |          |     (Target/LAN)      |
+-----------------------+          +-----------------------+          +-----------------------+
| Adapter: Bridged      |          | WAN: 192.168.1.10/24  |          | IP: 192.168.1.100/24  |
| IP: 192.168.1.x (DHCP)|--------->|      (Bridged)        |--------->| Gateway: 192.168.1.1  |
|                       |          | LAN: 192.168.1.1/24   |          | Adapter: Internal     |
|                       |          |      (Internal)       |          |                       |
+-----------------------+          +-----------------------+          +-----------------------+
      [Bridged Network]                                                  [Internal Network]
```

### Virtual Machine Specifications

| Role | Operating System | Network Adapter(s) | IP Configuration | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Firewall** | pfSense CE | **WAN:** Bridged Adapter<br>**LAN:** Internal Network | WAN: `192.168.1.10/24`<br>LAN: `192.168.1.1/24` | Perimeter filtering, NAT, rate-limiting, and IDS/IPS |
| **Target** | Ubuntu Server | **NIC 1:** Internal Network | IP: `192.168.1.100/24`<br>GW: `192.168.1.1` | Hosts target web and SSH services |
| **Attacker** | Kali Linux | **NIC 1:** Bridged Adapter | IP: `192.168.1.x/24` (DHCP/Static) | Reconnaissance, brute-forcing, and vulnerability scanning |

> **⚠️ Routing & Subnet Note:** Since both the Bridged WAN (`192.168.1.10/24`) and Internal LAN (`192.168.1.1/24`) share the `192.168.1.0/24` address space, ensure **Block private networks and loopback addresses** is unchecked on the WAN interface. If you encounter asymmetric routing between WAN and LAN, move the internal LAN to a separate subnet such as `10.0.1.0/24` (e.g., pfSense LAN `10.0.1.1` and Ubuntu `10.0.1.100`).

---

<img width="1920" height="1080" alt="Screenshot (371)" src="https://github.com/user-attachments/assets/1b4331cb-1e51-4639-a5d8-20de82ef7cc4" />

<img width="1920" height="1080" alt="Screenshot (372)" src="https://github.com/user-attachments/assets/6c4b5c97-c613-458f-8da1-9d4d13183502" />

<img width="1920" height="1080" alt="Screenshot (373)" src="https://github.com/user-attachments/assets/586be5d3-3248-4ee6-b29f-d3c57bda9ca7" />

## ⚙️ Lab Setup & Configuration

### 1. pfSense Firewall Configuration
1. **Interface Assignment:**
   * Assign the **Bridged Adapter** as **WAN** and set a static IPv4 address of `192.168.1.10/24`.
   * Assign the **Internal Network** adapter as **LAN** and set a static IPv4 address of `192.168.1.1/24`.
2. **Allow RFC1918 Traffic on WAN:**
   * Go to **Interfaces > WAN**.
   * Uncheck **Block private networks and loopback addresses** and click **Save / Apply Changes**.
3. **Port Forwarding (NAT) to Ubuntu:**
   * Go to **Firewall > NAT > Port Forward** and click **Add**.
   * **Interface:** `WAN` | **Protocol:** `TCP` | **Destination:** `WAN address` | **Destination Port:** `80` (HTTP)
   * **Redirect Target IP:** `192.168.1.100` | **Redirect Target Port:** `80`
   * Save and apply changes (pfSense will automatically create the corresponding WAN firewall rule).

<img width="1920" height="1080" alt="Screenshot (362)" src="https://github.com/user-attachments/assets/12c3b977-e4ef-4cb3-918f-dd61ac1f4cbd" />

### 2. Ubuntu Target Configuration
Configure the static IP on Ubuntu via Netplan (`/etc/netplan/00-installer-config.yaml`):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Apply the network configuration and start the target services:

```bash
sudo netplan apply
sudo apt update && sudo apt install -y apache2 openssh-server
```

---

<img width="1920" height="1080" alt="Screenshot (369)" src="https://github.com/user-attachments/assets/6fdce2bc-79c2-4267-ba1c-6a4e23ce5b15" />


## ⚔️ Attack & Defense Scenarios

### Scenario 1: Perimeter Reconnaissance (Nmap)
* **Attack (Kali Linux):** Scan the pfSense WAN IP (`192.168.1.10`) from the bridged network to enumerate exposed ports.
  ```bash
  nmap -sS -sV -Pn 192.168.1.10
  ```
* **Expected Result:** Port `80` shows as `open` (forwarded to `192.168.1.100`), while Ubuntu's SSH port (`22`) and internal services remain unreachable.
* **Firewall Logs:** Inspect blocked probes under **Status > System Logs > Firewall**.

### Scenario 2: Connection Rate-Limiting Against Brute Force
* **Attack (Kali Linux):** Launch an automated login brute-force attack against the forwarded web service using Hydra:
  ```bash
  hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.1.10 http-get /
  ```
* **Defense (pfSense):**
  1. Navigate to **Firewall > Rules > WAN** and edit the NAT rule for port `80`.
  2. Click **Display Advanced**.
  3. Set **Max src. conn. Rate** to `15` connections per `5` seconds.
  4. Save and apply. Kali's bridged IP is automatically placed into the `<virusprot>` block table once the threshold is exceeded.

### Scenario 3: Intrusion Detection & Prevention (Suricata)
* **Defense Setup:**
  1. Install **Suricata** from **System > Package Manager > Available Packages**.
  2. Configure Suricata on the **WAN** interface (`192.168.1.10`), enable **Emerging Threats Open** rules, and turn on **Block Offenders**.
* **Attack (Kali Linux):** Run a web vulnerability scan against the target:
  ```bash
  nikto -h [http://192.168.1.10](http://192.168.1.10)
  ```
* **Expected Result:** Suricata flags the Nikto scan signatures under **Services > Suricata > Alerts** and drops subsequent packets from Kali's IP.

---

<img width="1920" height="1080" alt="Screenshot (370)" src="https://github.com/user-attachments/assets/e988f886-3a34-4590-85d3-d2f5b449ad69" />

## 📁 Repository Structure

```text
├── README.md                    # Lab documentation and setup guide
├── diagrams/
│   └── topology.png             # Network architecture diagram
├── configs/
│   ├── pfsense-config.xml       # Exported pfSense NAT and firewall rules
│   └── 00-installer-config.yaml # Ubuntu Netplan static IP config
└── logs/
    ├── nmap-wan-scan.txt        # Nmap output against 192.168.1.10
    └── suricata-alerts.log      # Captured IDS/IPS alert logs
```

---

## ⚠️ Disclaimer
This repository is intended strictly for educational purposes and authorized security testing within an isolated lab environment. Never run offensive security tools against networks without explicit permission.
